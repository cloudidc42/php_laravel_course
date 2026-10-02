# Part 041: Laravel API Development

**ระดับ:** สูง / มืออาชีพ  
**เวลาเรียน:** 5-6 ชั่วโมง  
**ความต้องการก่อนเรียน:** Part 036-040, REST API concepts

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ออกแบบ RESTful API ได้ถูกต้อง
- ใช้ API Resources (Transformers) สำหรับ response formatting
- ตั้งค่า API Authentication ด้วย Sanctum
- ตั้งค่า Rate Limiting
- จัดการ API Versioning
- Workshop: สร้าง Full REST API สำหรับ Todo App

---

## 1. RESTful API Design Principles

### 1.1 HTTP Methods

| Method | Path | Action | Description |
|--------|------|--------|-------------|
| GET | /todos | index | ดูรายการทั้งหมด |
| POST | /todos | store | สร้างใหม่ |
| GET | /todos/{id} | show | ดูรายการเดียว |
| PUT/PATCH | /todos/{id} | update | แก้ไข |
| DELETE | /todos/{id} | destroy | ลบ |

### 1.2 HTTP Status Codes

```php
// Success
200 OK              - ดึงข้อมูลสำเร็จ
201 Created         - สร้างใหม่สำเร็จ
204 No Content      - ลบสำเร็จ (ไม่ return body)

// Client Error
400 Bad Request     - ข้อมูลผิดพลาด
401 Unauthorized    - ยังไม่ได้ login
403 Forbidden       - ไม่มีสิทธิ์
404 Not Found       - ไม่พบข้อมูล
422 Unprocessable   - validation error
429 Too Many Requests - rate limit exceeded

// Server Error
500 Internal Server Error
503 Service Unavailable
```

### 1.3 Routes

```php
// routes/api.php

use App\Http\Controllers\Api\TodoController;
use App\Http\Controllers\Api\AuthController;

// Public routes
Route::post('/auth/register', [AuthController::class, 'register']);
Route::post('/auth/login', [AuthController::class, 'login']);

// Protected routes
Route::middleware('auth:sanctum')->group(function () {
    Route::post('/auth/logout', [AuthController::class, 'logout']);
    Route::get('/auth/me', [AuthController::class, 'me']);
    
    // Todo CRUD
    Route::apiResource('todos', TodoController::class);
    
    // Nested resources
    Route::apiResource('todos.comments', TodoCommentController::class)
         ->shallow(); // ใช้ shallow ทำให้ routes ที่ต้องการเฉพาะ id ของ child ไม่ต้องมี parent
});
```

---

## 2. API Resources (Transformers)

API Resources ช่วย format response ก่อนส่งไปยัง client

```bash
php artisan make:resource TodoResource
php artisan make:resource TodoCollection
```

```php
<?php
// app/Http/Resources/TodoResource.php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class TodoResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'description' => $this->description,
            'status' => $this->status,
            'priority' => $this->priority,
            'due_date' => $this->due_date?->format('Y-m-d'),
            'is_completed' => $this->is_completed,
            'completed_at' => $this->completed_at?->toISOString(),
            
            // Relationship (โหลดเฉพาะเมื่อ eager loaded)
            'tags' => TagResource::collection($this->whenLoaded('tags')),
            'comments_count' => $this->whenCounted('comments'),
            
            // Conditional field (แสดงเฉพาะ admin)
            'internal_notes' => $this->when(
                $request->user()?->isAdmin(),
                $this->internal_notes
            ),
            
            // Computed fields
            'is_overdue' => $this->due_date?->isPast() && !$this->is_completed,
            
            // Timestamps
            'created_at' => $this->created_at->toISOString(),
            'updated_at' => $this->updated_at->toISOString(),
        ];
    }

    /**
     * Customize wrapper key
     */
    public static $wrap = 'todo';
}
```

### 2.1 Collection Resource

```php
<?php
// app/Http/Resources/TodoCollection.php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\ResourceCollection;

class TodoCollection extends ResourceCollection
{
    /**
     * Resource class สำหรับแต่ละ item
     */
    public $collects = TodoResource::class;

    public function toArray(Request $request): array
    {
        return [
            'data' => $this->collection,
            'meta' => [
                'total' => $this->total(),
                'per_page' => $this->perPage(),
                'current_page' => $this->currentPage(),
                'last_page' => $this->lastPage(),
                'completed_count' => $this->collection->where('is_completed', true)->count(),
                'pending_count' => $this->collection->where('is_completed', false)->count(),
            ],
        ];
    }
}
```

### 2.2 ใช้ Resource ใน Controller

```php
<?php
// app/Http/Controllers/Api/TodoController.php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Requests\Api\StoreTodoRequest;
use App\Http\Requests\Api\UpdateTodoRequest;
use App\Http\Resources\TodoCollection;
use App\Http\Resources\TodoResource;
use App\Models\Todo;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class TodoController extends Controller
{
    public function index(Request $request): TodoCollection
    {
        $todos = Todo::query()
            ->where('user_id', auth()->id())
            ->withCount('comments')
            ->with('tags')
            ->when($request->status, fn($q) => $q->where('status', $request->status))
            ->when($request->search, fn($q) => $q->where('title', 'like', "%{$request->search}%"))
            ->when($request->priority, fn($q) => $q->where('priority', $request->priority))
            ->when($request->sort_by, function ($q) use ($request) {
                $q->orderBy(
                    $request->sort_by,
                    $request->sort_order ?? 'asc'
                );
            }, fn($q) => $q->latest())
            ->paginate($request->per_page ?? 15);

        return new TodoCollection($todos);
    }

    public function store(StoreTodoRequest $request): JsonResponse
    {
        $todo = Todo::create([
            ...$request->validated(),
            'user_id' => auth()->id(),
        ]);

        if ($request->has('tags')) {
            $todo->tags()->sync($request->tags);
        }

        return (new TodoResource($todo->load('tags')))
            ->response()
            ->setStatusCode(201);
    }

    public function show(Todo $todo): TodoResource
    {
        $this->authorize('view', $todo);
        
        return new TodoResource($todo->load(['tags', 'comments'])->loadCount('comments'));
    }

    public function update(UpdateTodoRequest $request, Todo $todo): TodoResource
    {
        $this->authorize('update', $todo);
        
        $todo->update($request->validated());

        if ($request->has('tags')) {
            $todo->tags()->sync($request->tags);
        }

        return new TodoResource($todo->load('tags'));
    }

    public function destroy(Todo $todo): JsonResponse
    {
        $this->authorize('delete', $todo);
        
        $todo->delete();

        return response()->json(null, 204);
    }
}
```

---

## 3. API Authentication ด้วย Sanctum

### 3.1 ติดตั้ง Sanctum

```bash
composer require laravel/sanctum
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
php artisan migrate
```

```php
// app/Models/User.php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
}
```

```php
// app/Http/Kernel.php (Laravel 10)
'api' => [
    \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
    \Illuminate\Routing\Middleware\ThrottleRequests::class.':api',
    \Illuminate\Routing\Middleware\SubstituteBindings::class,
],
```

### 3.2 Auth Controller

```php
<?php
// app/Http/Controllers/Api/AuthController.php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Requests\Api\LoginRequest;
use App\Http\Requests\Api\RegisterRequest;
use App\Http\Resources\UserResource;
use App\Models\User;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\ValidationException;

class AuthController extends Controller
{
    public function register(RegisterRequest $request): JsonResponse
    {
        $user = User::create([
            'name' => $request->name,
            'email' => $request->email,
            'password' => Hash::make($request->password),
        ]);

        // สร้าง token พร้อม abilities
        $token = $user->createToken(
            $request->device_name ?? 'api',
            ['todo:read', 'todo:write'] // abilities
        );

        return response()->json([
            'user' => new UserResource($user),
            'token' => $token->plainTextToken,
            'token_type' => 'Bearer',
        ], 201);
    }

    public function login(LoginRequest $request): JsonResponse
    {
        $user = User::where('email', $request->email)->first();

        if (!$user || !Hash::check($request->password, $user->password)) {
            throw ValidationException::withMessages([
                'email' => ['Invalid credentials'],
            ]);
        }

        if (!$user->is_active) {
            return response()->json(['message' => 'Account is disabled'], 403);
        }

        // ลบ token เก่าของ device นี้
        $user->tokens()
             ->where('name', $request->device_name ?? 'api')
             ->delete();

        $token = $user->createToken(
            $request->device_name ?? 'api',
            ['*'] // ทุก ability
        );

        $user->update(['last_login_at' => now()]);

        return response()->json([
            'user' => new UserResource($user),
            'token' => $token->plainTextToken,
            'token_type' => 'Bearer',
            'expires_at' => now()->addDays(30)->toISOString(),
        ]);
    }

    public function logout(Request $request): JsonResponse
    {
        // ลบเฉพาะ token ปัจจุบัน
        $request->user()->currentAccessToken()->delete();

        return response()->json(['message' => 'Logged out successfully']);
    }

    public function logoutAll(Request $request): JsonResponse
    {
        // ลบทุก token
        $request->user()->tokens()->delete();

        return response()->json(['message' => 'Logged out from all devices']);
    }

    public function me(Request $request): JsonResponse
    {
        return response()->json(new UserResource($request->user()));
    }

    public function refresh(Request $request): JsonResponse
    {
        $user = $request->user();
        $currentToken = $user->currentAccessToken();
        
        // สร้าง token ใหม่
        $newToken = $user->createToken($currentToken->name, $currentToken->abilities);
        
        // ลบ token เก่า
        $currentToken->delete();

        return response()->json([
            'token' => $newToken->plainTextToken,
            'token_type' => 'Bearer',
        ]);
    }
}
```

---

## 4. Rate Limiting

### 4.1 ตั้งค่า Rate Limiter

```php
// app/Providers/RouteServiceProvider.php (Laravel 10)

use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

public function boot(): void
{
    // Rate limit สำหรับ API
    RateLimiter::for('api', function (Request $request) {
        return $request->user()
            ? Limit::perMinute(60)->by($request->user()->id)
            : Limit::perMinute(10)->by($request->ip());
    });

    // Rate limit สำหรับ login
    RateLimiter::for('login', function (Request $request) {
        return Limit::perMinute(5)
            ->by($request->ip())
            ->response(function () {
                return response()->json([
                    'message' => 'Too many login attempts. Please try again later.',
                    'retry_after' => 60,
                ], 429);
            });
    });

    // Rate limit ตาม plan
    RateLimiter::for('api-premium', function (Request $request) {
        $user = $request->user();
        
        return match($user?->plan) {
            'premium' => Limit::perMinute(300)->by($user->id),
            'basic' => Limit::perMinute(60)->by($user->id),
            default => Limit::perMinute(10)->by($request->ip()),
        };
    });
}
```

```php
// routes/api.php
Route::middleware(['auth:sanctum', 'throttle:api'])->group(function () {
    Route::apiResource('todos', TodoController::class);
});

Route::middleware('throttle:login')
     ->post('/auth/login', [AuthController::class, 'login']);
```

---

## 5. API Versioning

### 5.1 URL-based Versioning (แนะนำ)

```php
// routes/api.php

// Version 1
Route::prefix('v1')->namespace('App\Http\Controllers\Api\V1')->group(function () {
    Route::apiResource('todos', TodoController::class);
});

// Version 2 (ใหม่กว่า)
Route::prefix('v2')->namespace('App\Http\Controllers\Api\V2')->group(function () {
    Route::apiResource('todos', TodoController::class);
});
```

โครงสร้างไฟล์:
```
app/
  Http/
    Controllers/
      Api/
        V1/
          TodoController.php
          AuthController.php
        V2/
          TodoController.php    <- มีการเปลี่ยน logic
```

### 5.2 Header-based Versioning

```php
// app/Http/Middleware/ApiVersion.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class ApiVersion
{
    public function handle(Request $request, Closure $next, string $version): mixed
    {
        $requestedVersion = $request->header('API-Version', 'v1');
        
        if ($requestedVersion !== $version) {
            return response()->json([
                'error' => 'API version mismatch',
                'requested' => $requestedVersion,
                'available' => ['v1', 'v2'],
            ], 400);
        }

        return $next($request);
    }
}
```

---

## 6. Error Handling

```php
// app/Exceptions/Handler.php

use Illuminate\Auth\AuthenticationException;
use Illuminate\Validation\ValidationException;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;

class Handler extends ExceptionHandler
{
    public function render($request, Throwable $e): mixed
    {
        if ($request->expectsJson()) {
            return $this->handleApiException($request, $e);
        }

        return parent::render($request, $e);
    }

    private function handleApiException(Request $request, Throwable $e): JsonResponse
    {
        if ($e instanceof ValidationException) {
            return response()->json([
                'message' => 'Validation failed',
                'errors' => $e->errors(),
            ], 422);
        }

        if ($e instanceof AuthenticationException) {
            return response()->json([
                'message' => 'Unauthenticated',
            ], 401);
        }

        if ($e instanceof NotFoundHttpException) {
            return response()->json([
                'message' => 'Resource not found',
            ], 404);
        }

        if ($e instanceof \Illuminate\Auth\Access\AuthorizationException) {
            return response()->json([
                'message' => 'Forbidden',
            ], 403);
        }

        if ($e instanceof \Illuminate\Database\Eloquent\ModelNotFoundException) {
            $model = class_basename($e->getModel());
            return response()->json([
                'message' => "{$model} not found",
            ], 404);
        }

        // Production: ไม่แสดง error details
        if (app()->isProduction()) {
            return response()->json([
                'message' => 'Internal server error',
            ], 500);
        }

        return response()->json([
            'message' => $e->getMessage(),
            'exception' => get_class($e),
            'file' => $e->getFile(),
            'line' => $e->getLine(),
        ], 500);
    }
}
```

---

## 7. Workshop: Full REST API สำหรับ Todo App

### 7.1 Migrations

```php
// database/migrations/create_todos_table.php

Schema::create('todos', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->string('title');
    $table->text('description')->nullable();
    $table->enum('status', ['pending', 'in_progress', 'completed', 'cancelled'])
          ->default('pending');
    $table->enum('priority', ['low', 'medium', 'high'])->default('medium');
    $table->date('due_date')->nullable();
    $table->timestamp('completed_at')->nullable();
    $table->timestamps();
    $table->softDeletes();
    
    $table->index(['user_id', 'status']);
    $table->index(['user_id', 'due_date']);
});
```

### 7.2 Todo Model

```php
<?php
// app/Models/Todo.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\SoftDeletes;

class Todo extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'user_id', 'title', 'description',
        'status', 'priority', 'due_date', 'completed_at',
    ];

    protected $casts = [
        'due_date' => 'date',
        'completed_at' => 'datetime',
    ];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function getIsCompletedAttribute(): bool
    {
        return $this->status === 'completed';
    }

    public function complete(): void
    {
        $this->update([
            'status' => 'completed',
            'completed_at' => now(),
        ]);
    }

    public function reopen(): void
    {
        $this->update([
            'status' => 'pending',
            'completed_at' => null,
        ]);
    }

    // Scopes
    public function scopePending($query)
    {
        return $query->where('status', 'pending');
    }

    public function scopeOverdue($query)
    {
        return $query->where('due_date', '<', today())
                    ->whereNotIn('status', ['completed', 'cancelled']);
    }
}
```

### 7.3 Form Requests

```php
<?php
// app/Http/Requests/Api/StoreTodoRequest.php

namespace App\Http\Requests\Api;

use Illuminate\Foundation\Http\FormRequest;

class StoreTodoRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'title' => 'required|string|max:255',
            'description' => 'nullable|string|max:2000',
            'priority' => 'nullable|in:low,medium,high',
            'due_date' => 'nullable|date|after:today',
            'tags' => 'nullable|array',
            'tags.*' => 'integer|exists:tags,id',
        ];
    }
}
```

### 7.4 Policy

```php
<?php
// app/Policies/TodoPolicy.php

namespace App\Policies;

use App\Models\Todo;
use App\Models\User;

class TodoPolicy
{
    public function viewAny(User $user): bool
    {
        return true;
    }

    public function view(User $user, Todo $todo): bool
    {
        return $user->id === $todo->user_id;
    }

    public function create(User $user): bool
    {
        return true;
    }

    public function update(User $user, Todo $todo): bool
    {
        return $user->id === $todo->user_id;
    }

    public function delete(User $user, Todo $todo): bool
    {
        return $user->id === $todo->user_id;
    }
}
```

### 7.5 Routes ครบถ้วน

```php
// routes/api.php

use App\Http\Controllers\Api;

Route::prefix('v1')->group(function () {
    // Auth
    Route::post('/auth/register', [Api\AuthController::class, 'register']);
    Route::post('/auth/login', [Api\AuthController::class, 'login'])
         ->middleware('throttle:login');

    Route::middleware('auth:sanctum')->group(function () {
        // Auth
        Route::post('/auth/logout', [Api\AuthController::class, 'logout']);
        Route::post('/auth/logout-all', [Api\AuthController::class, 'logoutAll']);
        Route::get('/auth/me', [Api\AuthController::class, 'me']);
        Route::post('/auth/refresh', [Api\AuthController::class, 'refresh']);

        // Todos
        Route::apiResource('todos', Api\TodoController::class);
        Route::patch('/todos/{todo}/complete', [Api\TodoController::class, 'complete']);
        Route::patch('/todos/{todo}/reopen', [Api\TodoController::class, 'reopen']);
        
        // Stats
        Route::get('/stats/todos', [Api\StatController::class, 'todos']);
    });
});
```

### 7.6 ทดสอบด้วย cURL

```bash
# Register
curl -X POST http://localhost:8000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name":"Test User","email":"test@example.com","password":"password","password_confirmation":"password"}'

# Login
curl -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password"}'

# Get Todos (ต้องมี token)
curl -X GET http://localhost:8000/api/v1/todos \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"

# Create Todo
curl -X POST http://localhost:8000/api/v1/todos \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{"title":"Learn Laravel API","priority":"high","due_date":"2024-12-31"}'
```

---

## Quiz

### คำถาม 1
`$this->whenLoaded('tags')` ใน Resource ทำอะไร?

**A)** โหลด tags เสมอ  
**B)** แสดง tags เฉพาะเมื่อถูก eager loaded มาแล้ว  
**C)** ลบ tags  
**D)** นับ tags  

**เฉลย: B** - `whenLoaded()` ป้องกัน N+1 problem โดยแสดงข้อมูลเฉพาะเมื่อ relationship ถูก load มาแล้ว

---

### คำถาม 2
Sanctum Token มี Abilities ไว้ทำอะไร?

**A)** กำหนด expiry time  
**B)** กำหนดสิทธิ์ที่ token นี้ทำได้ (granular permissions)  
**C)** กำหนด rate limit  
**D)** encrypt token  

**เฉลย: B** - Abilities คือ granular permissions เช่น `['todo:read', 'todo:write']` ทำให้ token ทำได้เฉพาะที่กำหนด

---

### คำถาม 3
Rate Limiting `.by($request->user()->id)` ต่างจาก `.by($request->ip())` อย่างไร?

**A)** ไม่ต่างกัน  
**B)** by user ID: นับแยกต่อ user, by IP: นับแยกต่อ IP address  
**C)** by user ID เร็วกว่า  
**D)** by IP ปลอดภัยกว่า  

**เฉลย: B** - การนับแยกต่อ user ID ทำให้แต่ละ user มี limit ของตัวเอง ส่วน by IP ทุก request จาก IP เดียวกันนับรวมกัน

---

### คำถาม 4
`->shallow()` ใน `apiResource` ทำให้ routes เป็นอย่างไร?

**A)** ลด routes ให้น้อยลง  
**B)** Routes ที่ต้องการเฉพาะ child ID จะไม่ต้องมี parent prefix  
**C)** เพิ่ม middleware อัตโนมัติ  
**D)** Cache routes  

**เฉลย: B** - `shallow()` ทำให้ show, update, destroy ใช้เฉพาะ child ID (`/comments/{comment}`) แทนที่จะเป็น (`/todos/{todo}/comments/{comment}`)

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- ✅ RESTful API Design Principles และ HTTP Status Codes
- ✅ API Resources สำหรับ response formatting
- ✅ Sanctum Authentication พร้อม Token Abilities
- ✅ Rate Limiting แบบ per-user และ per-plan
- ✅ API Versioning
- ✅ Error Handling แบบ API
- ✅ Workshop: Full REST API สำหรับ Todo App

---

## ลิงก์ไป Part ถัดไป

➡️ **[Part 042: Laravel Testing](part-042-laravel-testing.md)**  
เรียนรู้เรื่อง PHPUnit, Feature Tests, Unit Tests, HTTP Tests และ Database Testing
