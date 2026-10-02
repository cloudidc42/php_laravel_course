# Part 028: Laravel Controllers

**ระดับ: กลาง (Intermediate)**

---

## เป้าหมายของ Part นี้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- สร้างและใช้งาน Controller ได้อย่างถูกต้อง
- เข้าใจ Resource Controller และ action ทั้ง 7
- สร้าง Single Action Controller
- ใช้ Middleware ใน Controller
- ใช้ Dependency Injection ใน Controller
- สร้าง BlogController ที่มี CRUD ครบสมบูรณ์

---

## 1. สร้าง Controller

### สร้างด้วย Artisan

```bash
# Controller พื้นฐาน
php artisan make:controller PostController

# Resource Controller (มี 7 methods)
php artisan make:controller PostController --resource

# API Resource Controller (ไม่มี create/edit)
php artisan make:controller PostController --api

# Invokable Controller (Single Action)
php artisan make:controller ShowDashboard --invokable

# Controller พร้อม Model (ไม่สร้าง resource methods)
php artisan make:controller PostController --model=Post

# Resource Controller พร้อม Model
php artisan make:controller PostController --resource --model=Post

# Resource Controller + Requests สำหรับ store/update
php artisan make:controller PostController --resource --model=Post --requests
```

### โครงสร้าง Controller พื้นฐาน

```php
// app/Http/Controllers/PostController.php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class PostController extends Controller
{
    public function index()
    {
        // แสดงรายการ
    }

    public function show(int $id)
    {
        // แสดงรายละเอียด
    }

    public function store(Request $request)
    {
        // บันทึกข้อมูล
    }
}
```

### Base Controller

```php
// app/Http/Controllers/Controller.php
<?php

namespace App\Http\Controllers;

abstract class Controller
{
    // Methods ที่ทุก Controller สืบทอด
    
    /**
     * Helper method สำหรับ success response
     */
    protected function successResponse(mixed $data = null, string $message = 'Success', int $status = 200)
    {
        return response()->json([
            'success' => true,
            'message' => $message,
            'data'    => $data,
        ], $status);
    }

    /**
     * Helper method สำหรับ error response
     */
    protected function errorResponse(string $message, int $status = 400, mixed $errors = null)
    {
        return response()->json([
            'success' => false,
            'message' => $message,
            'errors'  => $errors,
        ], $status);
    }
}
```

---

## 2. Resource Controller

Resource Controller มี 7 methods มาตรฐานสำหรับ CRUD

```php
// app/Http/Controllers/PostController.php
<?php

namespace App\Http\Controllers;

use App\Models\Post;
use App\Http\Requests\StorePostRequest;
use App\Http\Requests\UpdatePostRequest;
use Illuminate\Http\Request;

class PostController extends Controller
{
    /**
     * GET /posts
     * แสดงรายการ posts ทั้งหมด
     */
    public function index()
    {
        $posts = Post::with('author', 'category')
            ->latest()
            ->paginate(15);

        return view('posts.index', compact('posts'));
    }

    /**
     * GET /posts/create
     * แสดง form สร้าง post ใหม่
     */
    public function create()
    {
        $categories = \App\Models\Category::all();
        return view('posts.create', compact('categories'));
    }

    /**
     * POST /posts
     * บันทึก post ใหม่
     */
    public function store(StorePostRequest $request)
    {
        // StorePostRequest ทำ validation แล้ว
        $post = Post::create([
            'title'       => $request->title,
            'content'     => $request->content,
            'slug'        => \Illuminate\Support\Str::slug($request->title),
            'category_id' => $request->category_id,
            'user_id'     => auth()->id(),
        ]);

        // จัดการ tags (many-to-many)
        if ($request->has('tags')) {
            $post->tags()->sync($request->tags);
        }

        // จัดการ image upload
        if ($request->hasFile('image')) {
            $path = $request->file('image')->store('posts', 'public');
            $post->update(['image' => $path]);
        }

        return redirect()
            ->route('posts.show', $post)
            ->with('success', 'สร้างบทความสำเร็จ!');
    }

    /**
     * GET /posts/{post}
     * แสดงรายละเอียด post
     * ใช้ Route Model Binding
     */
    public function show(Post $post)
    {
        // เพิ่ม view count
        $post->increment('views');

        // โหลด relationships
        $post->load('author', 'category', 'tags', 'comments.user');

        // บทความที่เกี่ยวข้อง
        $relatedPosts = Post::where('category_id', $post->category_id)
            ->where('id', '!=', $post->id)
            ->published()
            ->limit(5)
            ->get();

        return view('posts.show', compact('post', 'relatedPosts'));
    }

    /**
     * GET /posts/{post}/edit
     * แสดง form แก้ไข post
     */
    public function edit(Post $post)
    {
        // ตรวจสอบสิทธิ์ - ใช้ Policy
        $this->authorize('update', $post);

        $categories = \App\Models\Category::all();
        $post->load('tags');

        return view('posts.edit', compact('post', 'categories'));
    }

    /**
     * PUT/PATCH /posts/{post}
     * อัปเดต post
     */
    public function update(UpdatePostRequest $request, Post $post)
    {
        $this->authorize('update', $post);

        $post->update($request->validated());

        if ($request->has('tags')) {
            $post->tags()->sync($request->tags);
        }

        if ($request->hasFile('image')) {
            // ลบรูปเก่า
            if ($post->image) {
                \Illuminate\Support\Facades\Storage::disk('public')->delete($post->image);
            }
            $path = $request->file('image')->store('posts', 'public');
            $post->update(['image' => $path]);
        }

        return redirect()
            ->route('posts.show', $post)
            ->with('success', 'อัปเดตบทความสำเร็จ!');
    }

    /**
     * DELETE /posts/{post}
     * ลบ post
     */
    public function destroy(Post $post)
    {
        $this->authorize('delete', $post);

        // ลบไฟล์
        if ($post->image) {
            \Illuminate\Support\Facades\Storage::disk('public')->delete($post->image);
        }

        $post->delete();

        return redirect()
            ->route('posts.index')
            ->with('success', 'ลบบทความสำเร็จ!');
    }
}
```

---

## 3. Form Request Validation

แทนที่จะ validate ใน Controller ให้แยกออกมาเป็น Form Request

```bash
php artisan make:request StorePostRequest
php artisan make:request UpdatePostRequest
```

```php
// app/Http/Requests/StorePostRequest.php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class StorePostRequest extends FormRequest
{
    /**
     * กำหนดว่าใครมีสิทธิ์ทำ request นี้ได้
     */
    public function authorize(): bool
    {
        return auth()->check(); // ต้อง login
    }

    /**
     * Validation rules
     */
    public function rules(): array
    {
        return [
            'title'       => 'required|string|max:255',
            'content'     => 'required|string|min:100',
            'category_id' => 'required|exists:categories,id',
            'image'       => 'nullable|image|mimes:jpg,png,webp|max:2048',
            'tags'        => 'nullable|array',
            'tags.*'      => 'exists:tags,id',
            'status'      => ['required', Rule::in(['draft', 'published'])],
        ];
    }

    /**
     * Custom error messages ภาษาไทย
     */
    public function messages(): array
    {
        return [
            'title.required'       => 'กรุณากรอกชื่อบทความ',
            'title.max'            => 'ชื่อบทความต้องไม่เกิน 255 ตัวอักษร',
            'content.required'     => 'กรุณากรอกเนื้อหาบทความ',
            'content.min'          => 'เนื้อหาต้องมีอย่างน้อย 100 ตัวอักษร',
            'category_id.required' => 'กรุณาเลือกหมวดหมู่',
            'category_id.exists'   => 'หมวดหมู่ที่เลือกไม่ถูกต้อง',
            'image.image'          => 'ไฟล์ต้องเป็นรูปภาพ',
            'image.max'            => 'รูปภาพต้องมีขนาดไม่เกิน 2MB',
        ];
    }

    /**
     * Custom attribute names
     */
    public function attributes(): array
    {
        return [
            'title'       => 'ชื่อบทความ',
            'content'     => 'เนื้อหา',
            'category_id' => 'หมวดหมู่',
        ];
    }

    /**
     * Prepare data ก่อน validate
     */
    protected function prepareForValidation(): void
    {
        $this->merge([
            'title' => trim($this->title),
        ]);
    }
}
```

```php
// app/Http/Requests/UpdatePostRequest.php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class UpdatePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        // ตรวจสอบว่า user เป็นเจ้าของ post
        return $this->route('post')->user_id === auth()->id();
    }

    public function rules(): array
    {
        return [
            'title'       => 'sometimes|required|string|max:255',
            'content'     => 'sometimes|required|string|min:100',
            'category_id' => 'sometimes|required|exists:categories,id',
            'image'       => 'nullable|image|mimes:jpg,png,webp|max:2048',
            'tags'        => 'nullable|array',
            'tags.*'      => 'exists:tags,id',
            'status'      => ['sometimes', Rule::in(['draft', 'published'])],
        ];
    }
}
```

---

## 4. Single Action Controller

เหมาะสำหรับ action ที่ซับซ้อนแต่ทำงานอย่างเดียว

```bash
php artisan make:controller ProcessPayment --invokable
```

```php
// app/Http/Controllers/ProcessPayment.php
<?php

namespace App\Http\Controllers;

use App\Models\Order;
use App\Services\PaymentService;
use Illuminate\Http\Request;
use Illuminate\Http\RedirectResponse;

class ProcessPayment extends Controller
{
    public function __construct(
        private readonly PaymentService $paymentService
    ) {}

    /**
     * __invoke ถูกเรียกเมื่อ invoke class เป็น function
     */
    public function __invoke(Request $request, Order $order): RedirectResponse
    {
        $request->validate([
            'card_token' => 'required|string',
        ]);

        try {
            $result = $this->paymentService->charge(
                order: $order,
                token: $request->card_token
            );

            if ($result->successful()) {
                $order->markAsPaid($result->transactionId);
                return redirect()
                    ->route('orders.show', $order)
                    ->with('success', 'ชำระเงินสำเร็จ!');
            }

            return back()->with('error', 'การชำระเงินไม่สำเร็จ: ' . $result->message);

        } catch (\Exception $e) {
            report($e);
            return back()->with('error', 'เกิดข้อผิดพลาด กรุณาลองใหม่อีกครั้ง');
        }
    }
}
```

```php
// routes/web.php
Route::post('/orders/{order}/payment', ProcessPayment::class)
    ->middleware(['auth'])
    ->name('orders.payment');
```

### ตัวอย่าง Single Action Controllers เพิ่มเติม

```php
// app/Http/Controllers/ShowDashboard.php
<?php

namespace App\Http\Controllers;

use App\Models\Post;
use App\Models\User;
use App\Models\Comment;

class ShowDashboard extends Controller
{
    public function __invoke()
    {
        return view('dashboard', [
            'totalPosts'    => Post::count(),
            'publishedPosts'=> Post::published()->count(),
            'totalUsers'    => User::count(),
            'totalComments' => Comment::count(),
            'recentPosts'   => Post::latest()->limit(5)->get(),
        ]);
    }
}
```

```php
// app/Http/Controllers/ExportPosts.php
<?php

namespace App\Http\Controllers;

use App\Models\Post;
use Illuminate\Http\Response;
use League\Csv\Writer;

class ExportPosts extends Controller
{
    public function __invoke(): Response
    {
        $posts = Post::with('author', 'category')->get();

        $csv = Writer::createFromString('');
        $csv->insertOne(['ID', 'Title', 'Author', 'Category', 'Published At']);

        foreach ($posts as $post) {
            $csv->insertOne([
                $post->id,
                $post->title,
                $post->author->name,
                $post->category->name,
                $post->published_at?->format('d/m/Y'),
            ]);
        }

        return response($csv->getContent(), 200, [
            'Content-Type'        => 'text/csv',
            'Content-Disposition' => 'attachment; filename="posts.csv"',
        ]);
    }
}
```

---

## 5. Controller Middleware

### กำหนด Middleware ใน Constructor

```php
<?php

namespace App\Http\Controllers;

use App\Models\Post;
use Illuminate\Http\Request;

class PostController extends Controller
{
    public function __construct()
    {
        // ทุก method ต้อง auth
        $this->middleware('auth');

        // เฉพาะ create, store, edit, update, destroy
        $this->middleware('auth')->only(['create', 'store', 'edit', 'update', 'destroy']);

        // ยกเว้น index, show
        $this->middleware('auth')->except(['index', 'show']);

        // Middleware พร้อม parameter
        $this->middleware('throttle:10,1')->only(['store']);

        // Custom middleware
        $this->middleware('can:manage-posts')->only(['create', 'store', 'edit', 'update', 'destroy']);
    }
}
```

### กำหนด Middleware ด้วย Attribute (Laravel 11+)

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Routing\Controllers\HasMiddleware;
use Illuminate\Routing\Controllers\Middleware;

class PostController extends Controller implements HasMiddleware
{
    /**
     * กำหนด middleware สำหรับ controller
     */
    public static function middleware(): array
    {
        return [
            'auth',
            new Middleware('throttle:10,1', only: ['store']),
            new Middleware('can:update,post', only: ['edit', 'update']),
            new Middleware('can:delete,post', only: ['destroy']),
        ];
    }
}
```

---

## 6. Dependency Injection

Laravel's Service Container จัดการ dependencies ให้อัตโนมัติ

### Constructor Injection

```php
// app/Services/PostService.php
<?php

namespace App\Services;

use App\Models\Post;
use App\Models\Tag;
use Illuminate\Support\Str;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;

class PostService
{
    public function create(array $data, ?UploadedFile $image = null): Post
    {
        $data['slug'] = Str::slug($data['title']);
        $data['user_id'] = auth()->id();

        $post = Post::create($data);

        if ($image) {
            $path = $image->store('posts', 'public');
            $post->update(['image' => $path]);
        }

        if (isset($data['tags'])) {
            $post->tags()->sync($data['tags']);
        }

        return $post;
    }

    public function update(Post $post, array $data, ?UploadedFile $image = null): Post
    {
        $post->update($data);

        if ($image) {
            if ($post->image) {
                Storage::disk('public')->delete($post->image);
            }
            $path = $image->store('posts', 'public');
            $post->update(['image' => $path]);
        }

        if (isset($data['tags'])) {
            $post->tags()->sync($data['tags']);
        }

        return $post->fresh();
    }

    public function delete(Post $post): bool
    {
        if ($post->image) {
            Storage::disk('public')->delete($post->image);
        }

        return $post->delete();
    }

    public function publish(Post $post): Post
    {
        $post->update([
            'status'       => 'published',
            'published_at' => now(),
        ]);

        return $post;
    }
}
```

```php
// app/Http/Controllers/PostController.php
<?php

namespace App\Http\Controllers;

use App\Models\Post;
use App\Services\PostService;
use App\Http\Requests\StorePostRequest;
use App\Http\Requests\UpdatePostRequest;

class PostController extends Controller
{
    /**
     * Constructor Injection - Laravel inject PostService ให้อัตโนมัติ
     */
    public function __construct(
        private readonly PostService $postService
    ) {}

    public function index()
    {
        $posts = Post::with('author', 'category')->latest()->paginate(15);
        return view('posts.index', compact('posts'));
    }

    public function store(StorePostRequest $request)
    {
        $post = $this->postService->create(
            data: $request->validated(),
            image: $request->file('image')
        );

        return redirect()
            ->route('posts.show', $post)
            ->with('success', 'สร้างบทความสำเร็จ!');
    }

    public function update(UpdatePostRequest $request, Post $post)
    {
        $this->postService->update(
            post: $post,
            data: $request->validated(),
            image: $request->file('image')
        );

        return redirect()
            ->route('posts.show', $post)
            ->with('success', 'อัปเดตสำเร็จ!');
    }

    public function destroy(Post $post)
    {
        $this->authorize('delete', $post);
        $this->postService->delete($post);

        return redirect()
            ->route('posts.index')
            ->with('success', 'ลบสำเร็จ!');
    }
}
```

### Method Injection

```php
<?php

namespace App\Http\Controllers;

use App\Services\ReportService;
use App\Services\NotificationService;
use Illuminate\Http\Request;

class ReportController extends Controller
{
    /**
     * Method injection - inject ที่ method โดยตรง
     * ใช้เมื่อต้องการ service เฉพาะใน method นั้น
     */
    public function generate(
        Request $request,
        ReportService $reportService,        // Injected
        NotificationService $notifications    // Injected
    ) {
        $report = $reportService->generate(
            type: $request->type,
            from: $request->from_date,
            to: $request->to_date
        );

        $notifications->notify(
            user: auth()->user(),
            message: 'รายงานพร้อมแล้ว'
        );

        return response()->download($report->path);
    }
}
```

### Binding ใน Service Provider

```php
// app/Providers/AppServiceProvider.php
<?php

namespace App\Providers;

use App\Services\PostService;
use App\Services\CachedPostService;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Bind interface ไป implementation
        $this->app->bind(
            \App\Contracts\PostServiceContract::class,
            \App\Services\PostService::class
        );

        // Singleton - สร้างแค่ครั้งเดียว
        $this->app->singleton(PostService::class, function ($app) {
            return new PostService(
                cache: $app->make(\Illuminate\Cache\Repository::class)
            );
        });
    }
}
```

---

## 7. API Controller

```php
// app/Http/Controllers/Api/PostController.php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\Post;
use App\Http\Requests\StorePostRequest;
use App\Http\Requests\UpdatePostRequest;
use App\Http\Resources\PostResource;
use App\Http\Resources\PostCollection;
use App\Services\PostService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Response;

class PostController extends Controller
{
    public function __construct(
        private readonly PostService $postService
    ) {}

    /**
     * GET /api/posts
     */
    public function index(): PostCollection
    {
        $posts = Post::with('author', 'category', 'tags')
            ->filter(request()->only(['search', 'category', 'status']))
            ->latest()
            ->paginate(15);

        return new PostCollection($posts);
    }

    /**
     * POST /api/posts
     */
    public function store(StorePostRequest $request): PostResource
    {
        $post = $this->postService->create($request->validated());

        return (new PostResource($post))
            ->response()
            ->setStatusCode(201);
    }

    /**
     * GET /api/posts/{post}
     */
    public function show(Post $post): PostResource
    {
        $post->load('author', 'category', 'tags', 'comments');
        return new PostResource($post);
    }

    /**
     * PUT /api/posts/{post}
     */
    public function update(UpdatePostRequest $request, Post $post): PostResource
    {
        $post = $this->postService->update($post, $request->validated());
        return new PostResource($post);
    }

    /**
     * DELETE /api/posts/{post}
     */
    public function destroy(Post $post): Response
    {
        $this->authorize('delete', $post);
        $this->postService->delete($post);

        return response()->noContent(); // 204 No Content
    }
}
```

### API Resources

```bash
php artisan make:resource PostResource
php artisan make:resource PostCollection
```

```php
// app/Http/Resources/PostResource.php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class PostResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'         => $this->id,
            'title'      => $this->title,
            'slug'       => $this->slug,
            'content'    => $this->content,
            'excerpt'    => $this->excerpt,
            'image_url'  => $this->image ? asset('storage/' . $this->image) : null,
            'status'     => $this->status,
            'views'      => $this->views,
            
            // Relationships (โหลดเมื่อมี)
            'author'     => new UserResource($this->whenLoaded('author')),
            'category'   => new CategoryResource($this->whenLoaded('category')),
            'tags'       => TagResource::collection($this->whenLoaded('tags')),
            'comments'   => CommentResource::collection($this->whenLoaded('comments')),
            
            // Counts
            'comments_count' => $this->whenCounted('comments'),
            
            // Timestamps
            'published_at' => $this->published_at?->toISOString(),
            'created_at'   => $this->created_at->toISOString(),
            'updated_at'   => $this->updated_at->toISOString(),
            
            // Conditional fields
            'admin_notes' => $this->when(
                auth()->user()?->isAdmin(),
                $this->admin_notes
            ),
        ];
    }
}
```

---

## Workshop: สร้าง BlogController ครบ CRUD

### โจทย์

สร้าง Blog application ที่มี:
1. แสดงรายการบทความพร้อม pagination และ search
2. สร้างบทความใหม่
3. อ่านบทความ
4. แก้ไขบทความ (เฉพาะเจ้าของ)
5. ลบบทความ (เฉพาะเจ้าของ)
6. Publish/Unpublish บทความ

### Step 1: Migration

```php
// database/migrations/xxxx_create_posts_table.php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('posts', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->foreignId('category_id')->nullable()->constrained()->nullOnDelete();
            $table->string('title');
            $table->string('slug')->unique();
            $table->text('excerpt')->nullable();
            $table->longText('content');
            $table->string('image')->nullable();
            $table->enum('status', ['draft', 'published'])->default('draft');
            $table->unsignedInteger('views')->default(0);
            $table->timestamp('published_at')->nullable();
            $table->timestamps();
            $table->softDeletes();
            
            $table->index(['status', 'published_at']);
            $table->fullText(['title', 'content']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('posts');
    }
};
```

### Step 2: Model

```php
// app/Models/Post.php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Support\Str;

class Post extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'title', 'slug', 'excerpt', 'content', 'image',
        'status', 'published_at', 'user_id', 'category_id',
    ];

    protected $casts = [
        'published_at' => 'datetime',
        'views' => 'integer',
    ];

    // Auto-generate slug เมื่อ title เปลี่ยน
    protected static function boot(): void
    {
        parent::boot();

        static::creating(function (Post $post) {
            if (empty($post->slug)) {
                $post->slug = Str::slug($post->title);
            }
        });
    }

    // Route Model Binding ใช้ slug
    public function getRouteKeyName(): string
    {
        return 'slug';
    }

    // Relationships
    public function author()
    {
        return $this->belongsTo(User::class, 'user_id');
    }

    public function category()
    {
        return $this->belongsTo(Category::class);
    }

    public function tags()
    {
        return $this->belongsToMany(Tag::class);
    }

    public function comments()
    {
        return $this->hasMany(Comment::class)->latest();
    }

    // Scopes
    public function scopePublished(Builder $query): Builder
    {
        return $query->where('status', 'published');
    }

    public function scopeFilter(Builder $query, array $filters): Builder
    {
        return $query
            ->when($filters['search'] ?? null, function ($q, $search) {
                $q->whereFullText(['title', 'content'], $search);
            })
            ->when($filters['category'] ?? null, function ($q, $category) {
                $q->whereHas('category', fn($q) => $q->where('slug', $category));
            })
            ->when($filters['status'] ?? null, function ($q, $status) {
                $q->where('status', $status);
            });
    }

    // Accessors
    public function getExcerptAttribute($value): string
    {
        return $value ?: Str::limit(strip_tags($this->content), 150);
    }

    public function getReadTimeAttribute(): int
    {
        $wordCount = str_word_count(strip_tags($this->content));
        return max(1, (int) ceil($wordCount / 200)); // 200 words/minute
    }
}
```

### Step 3: BlogController

```php
// app/Http/Controllers/BlogController.php
<?php

namespace App\Http\Controllers;

use App\Models\Post;
use App\Models\Category;
use App\Http\Requests\StorePostRequest;
use App\Http\Requests\UpdatePostRequest;
use Illuminate\Http\Request;
use Illuminate\Http\RedirectResponse;
use Illuminate\View\View;
use Illuminate\Support\Facades\Storage;

class BlogController extends Controller
{
    public function __construct()
    {
        $this->middleware('auth')->except(['index', 'show']);
    }

    /**
     * GET /blog
     * รายการบทความสำหรับผู้อ่าน (เฉพาะ published)
     */
    public function index(Request $request): View
    {
        $posts = Post::with('author', 'category')
            ->published()
            ->filter($request->only(['search', 'category']))
            ->latest('published_at')
            ->paginate(12)
            ->withQueryString(); // เก็บ query string ใน pagination links

        $categories = Category::withCount(['posts' => fn($q) => $q->published()])
            ->having('posts_count', '>', 0)
            ->get();

        return view('blog.index', compact('posts', 'categories'));
    }

    /**
     * GET /blog/{post:slug}
     * อ่านบทความ
     */
    public function show(Post $post): View
    {
        // ถ้า draft และไม่ใช่เจ้าของ ให้ 404
        if ($post->status === 'draft' && auth()->id() !== $post->user_id) {
            abort(404);
        }

        $post->increment('views');
        $post->load('author', 'category', 'tags', 'comments.author');

        $relatedPosts = Post::where('category_id', $post->category_id)
            ->where('id', '!=', $post->id)
            ->published()
            ->latest('published_at')
            ->limit(4)
            ->get();

        return view('blog.show', compact('post', 'relatedPosts'));
    }

    /**
     * GET /blog/create
     */
    public function create(): View
    {
        $categories = Category::all();
        return view('blog.create', compact('categories'));
    }

    /**
     * POST /blog
     */
    public function store(StorePostRequest $request): RedirectResponse
    {
        $data = $request->validated();

        if ($request->hasFile('image')) {
            $data['image'] = $request->file('image')->store('posts', 'public');
        }

        if ($data['status'] === 'published') {
            $data['published_at'] = now();
        }

        $post = auth()->user()->posts()->create($data);

        if ($request->has('tags')) {
            $post->tags()->sync($request->tags);
        }

        return redirect()
            ->route('blog.show', $post)
            ->with('success', 'สร้างบทความเรียบร้อย!');
    }

    /**
     * GET /blog/{post:slug}/edit
     */
    public function edit(Post $post): View
    {
        $this->authorize('update', $post);

        $categories = Category::all();
        $post->load('tags');

        return view('blog.edit', compact('post', 'categories'));
    }

    /**
     * PUT /blog/{post:slug}
     */
    public function update(UpdatePostRequest $request, Post $post): RedirectResponse
    {
        $this->authorize('update', $post);

        $data = $request->validated();

        if ($request->hasFile('image')) {
            if ($post->image) {
                Storage::disk('public')->delete($post->image);
            }
            $data['image'] = $request->file('image')->store('posts', 'public');
        }

        // Set published_at เมื่อ publish ครั้งแรก
        if ($data['status'] === 'published' && !$post->published_at) {
            $data['published_at'] = now();
        }

        $post->update($data);

        if ($request->has('tags')) {
            $post->tags()->sync($request->tags);
        }

        return redirect()
            ->route('blog.show', $post)
            ->with('success', 'อัปเดตบทความเรียบร้อย!');
    }

    /**
     * DELETE /blog/{post:slug}
     */
    public function destroy(Post $post): RedirectResponse
    {
        $this->authorize('delete', $post);

        if ($post->image) {
            Storage::disk('public')->delete($post->image);
        }

        $post->delete();

        return redirect()
            ->route('blog.index')
            ->with('success', 'ลบบทความเรียบร้อย!');
    }

    /**
     * PATCH /blog/{post:slug}/publish
     * Toggle publish status
     */
    public function publish(Post $post): RedirectResponse
    {
        $this->authorize('update', $post);

        if ($post->status === 'published') {
            $post->update(['status' => 'draft', 'published_at' => null]);
            $message = 'ซ่อนบทความแล้ว';
        } else {
            $post->update(['status' => 'published', 'published_at' => now()]);
            $message = 'เผยแพร่บทความแล้ว';
        }

        return back()->with('success', $message);
    }
}
```

### Step 4: Routes

```php
// routes/web.php
Route::prefix('blog')->name('blog.')->group(function () {
    Route::get('/', [BlogController::class, 'index'])->name('index');
    Route::get('/create', [BlogController::class, 'create'])->name('create');
    Route::post('/', [BlogController::class, 'store'])->name('store');
    Route::get('/{post:slug}', [BlogController::class, 'show'])->name('show');
    Route::get('/{post:slug}/edit', [BlogController::class, 'edit'])->name('edit');
    Route::put('/{post:slug}', [BlogController::class, 'update'])->name('update');
    Route::delete('/{post:slug}', [BlogController::class, 'destroy'])->name('destroy');
    Route::patch('/{post:slug}/publish', [BlogController::class, 'publish'])->name('publish');
});
```

### Step 5: Policy

```bash
php artisan make:policy PostPolicy --model=Post
```

```php
// app/Policies/PostPolicy.php
<?php

namespace App\Policies;

use App\Models\Post;
use App\Models\User;

class PostPolicy
{
    public function update(User $user, Post $post): bool
    {
        return $user->id === $post->user_id || $user->isAdmin();
    }

    public function delete(User $user, Post $post): bool
    {
        return $user->id === $post->user_id || $user->isAdmin();
    }
}
```

---

## Quiz

### คำถาม

**1.** คำสั่งใดสร้าง Resource Controller พร้อม Form Requests?

a) `php artisan make:controller PostController --resource`
b) `php artisan make:controller PostController --resource --model=Post --requests`
c) `php artisan make:controller PostController --with-requests`
d) `php artisan make:controller PostController --full`

**2.** Single Action Controller ใช้ method ชื่ออะไร?

a) `handle()`
b) `execute()`
c) `__invoke()`
d) `run()`

**3.** `$this->authorize('update', $post)` ทำงานอย่างไร?

a) ตรวจสอบ middleware
b) ตรวจสอบผ่าน Policy และ throw 403 ถ้าไม่ผ่าน
c) ตรวจสอบ role
d) ตรวจสอบ permission ใน database

**4.** Form Request `authorize()` ที่ return `false` จะเกิดอะไร?

a) redirect ไป login
b) return 404
c) return 403 Forbidden
d) return 500

**5.** Constructor Injection ใน Controller คืออะไร?

a) inject Request object
b) inject dependencies ผ่าน constructor parameter
c) inject middleware
d) inject database connection

### เฉลย

| ข้อ | คำตอบ | อธิบาย |
|-----|-------|--------|
| 1 | **b** | `--requests` สร้าง StorePostRequest และ UpdatePostRequest พร้อมกัน |
| 2 | **c** | `__invoke()` ทำให้ class สามารถ invoke ได้เหมือน function |
| 3 | **b** | `authorize()` ตรวจสอบผ่าน Policy และ throw `AuthorizationException` (403) |
| 4 | **c** | FormRequest ที่ `authorize()` return false จะ return 403 |
| 5 | **b** | Constructor Injection ส่ง dependencies ผ่าน constructor parameters |

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- การสร้าง Controller ด้วย Artisan commands ต่างๆ
- Resource Controller และ 7 standard methods
- Form Request สำหรับ validation แยกจาก Controller
- Single Action Controller สำหรับ logic เดียว
- Controller Middleware ทั้งแบบ constructor และ attribute
- Dependency Injection ใน constructor และ method
- API Controller พร้อม API Resources
- Workshop: BlogController ที่มี CRUD ครบสมบูรณ์

---

## ลิงก์ไป Part ถัดไป

➡️ **[Part 029: Laravel Blade Templates](./part-029-laravel-blade.md)**

ใน Part ถัดไปเราจะเรียนรู้ Blade templating engine ของ Laravel ตั้งแต่ syntax พื้นฐาน Template inheritance, Components, จนถึง Custom Directives
