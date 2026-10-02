# Part 042: Laravel Testing

**ระดับ:** สูง / มืออาชีพ  
**เวลาเรียน:** 5-6 ชั่วโมง  
**ความต้องการก่อนเรียน:** Part 041, PHP OOP พื้นฐาน

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เขียน Feature Tests และ Unit Tests ด้วย PHPUnit
- ทดสอบ HTTP endpoints
- ทดสอบ Database operations
- Mock services และ external APIs
- Workshop: เขียน Tests ครบถ้วนสำหรับ Blog API

---

## 1. การตั้งค่า Testing Environment

### 1.1 phpunit.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="vendor/phpunit/phpunit/phpunit.xsd"
         bootstrap="vendor/autoload.php"
         colors="true">
    <testsuites>
        <testsuite name="Unit">
            <directory suffix="Test.php">./tests/Unit</directory>
        </testsuite>
        <testsuite name="Feature">
            <directory suffix="Test.php">./tests/Feature</directory>
        </testsuite>
    </testsuites>
    <source>
        <include>
            <directory suffix=".php">./app</directory>
        </include>
    </source>
    <php>
        <env name="APP_ENV" value="testing"/>
        <env name="BCRYPT_ROUNDS" value="4"/>       <!-- เร็วขึ้น -->
        <env name="CACHE_DRIVER" value="array"/>
        <env name="DB_CONNECTION" value="sqlite"/>
        <env name="DB_DATABASE" value=":memory:"/>   <!-- In-memory DB -->
        <env name="MAIL_MAILER" value="array"/>      <!-- ไม่ส่งจริง -->
        <env name="QUEUE_CONNECTION" value="sync"/>  <!-- รันทันที -->
        <env name="SESSION_DRIVER" value="array"/>
    </php>
</phpunit>
```

### 1.2 .env.testing

```env
APP_ENV=testing
DB_CONNECTION=sqlite
DB_DATABASE=:memory:
MAIL_MAILER=array
QUEUE_CONNECTION=sync
CACHE_DRIVER=array
```

### 1.3 รัน Tests

```bash
# รัน tests ทั้งหมด
php artisan test

# รันแบบ verbose
php artisan test --verbose

# รันเฉพาะ test file
php artisan test tests/Feature/TodoTest.php

# รันเฉพาะ test method
php artisan test --filter test_user_can_create_todo

# รัน test suite
php artisan test --testsuite=Feature

# ดู coverage
php artisan test --coverage
php artisan test --coverage --min=80  # ต้องมี coverage อย่างน้อย 80%

# รันขนานกัน (เร็วกว่า)
php artisan test --parallel
```

---

## 2. Unit Tests

Unit Tests ทดสอบ logic ของ class/method เดียวแบบ isolated

### 2.1 โครงสร้างพื้นฐาน

```php
<?php
// tests/Unit/Services/PriceCalculatorTest.php

namespace Tests\Unit\Services;

use App\Services\PriceCalculator;
use PHPUnit\Framework\TestCase; // ใช้ PHPUnit base, ไม่ใช่ Laravel TestCase

class PriceCalculatorTest extends TestCase
{
    private PriceCalculator $calculator;

    protected function setUp(): void
    {
        parent::setUp();
        $this->calculator = new PriceCalculator();
    }

    /** @test */
    public function it_calculates_discount_correctly(): void
    {
        $price = 1000;
        $discountPercent = 20;
        
        $result = $this->calculator->applyDiscount($price, $discountPercent);
        
        $this->assertEquals(800, $result);
    }

    /** @test */
    public function it_applies_minimum_price_floor(): void
    {
        $price = 10;
        $discountPercent = 99;
        
        $result = $this->calculator->applyDiscount($price, $discountPercent);
        
        // ราคาต่ำสุดคือ 1 บาท
        $this->assertGreaterThanOrEqual(1, $result);
    }

    /** @test */
    public function it_throws_exception_for_invalid_discount(): void
    {
        $this->expectException(\InvalidArgumentException::class);
        $this->expectExceptionMessage('Discount cannot exceed 100%');
        
        $this->calculator->applyDiscount(1000, 110);
    }

    /**
     * @test
     * @dataProvider priceProvider
     */
    public function it_calculates_tax_correctly(
        float $price,
        float $taxRate,
        float $expectedTotal
    ): void {
        $result = $this->calculator->withTax($price, $taxRate);
        
        $this->assertEqualsWithDelta($expectedTotal, $result, 0.01);
    }

    public static function priceProvider(): array
    {
        return [
            'standard VAT' => [100, 7, 107],
            'zero VAT' => [100, 0, 100],
            'high tax' => [1000, 15, 1150],
        ];
    }
}
```

### 2.2 Unit Test สำหรับ Model

```php
<?php
// tests/Unit/Models/TodoTest.php

namespace Tests\Unit\Models;

use App\Models\Todo;
use PHPUnit\Framework\TestCase;
use Carbon\Carbon;

class TodoTest extends TestCase
{
    /** @test */
    public function it_knows_when_overdue(): void
    {
        $todo = new Todo([
            'due_date' => Carbon::yesterday(),
            'status' => 'pending',
        ]);

        $this->assertTrue($todo->isOverdue());
    }

    /** @test */
    public function completed_todo_is_not_overdue(): void
    {
        $todo = new Todo([
            'due_date' => Carbon::yesterday(),
            'status' => 'completed',
        ]);

        $this->assertFalse($todo->isOverdue());
    }

    /** @test */
    public function it_formats_priority_label_correctly(): void
    {
        $cases = [
            'low' => 'ความสำคัญต่ำ',
            'medium' => 'ความสำคัญปานกลาง',
            'high' => 'ความสำคัญสูง',
        ];

        foreach ($cases as $priority => $expected) {
            $todo = new Todo(['priority' => $priority]);
            $this->assertEquals($expected, $todo->priority_label);
        }
    }
}
```

---

## 3. Feature Tests (HTTP Tests)

Feature Tests ทดสอบ endpoints ทั้งหมด รวม routing, middleware, database

### 3.1 พื้นฐาน

```php
<?php
// tests/Feature/Auth/AuthTest.php

namespace Tests\Feature\Auth;

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class AuthTest extends TestCase
{
    use RefreshDatabase; // Reset database หลังแต่ละ test

    /** @test */
    public function user_can_register(): void
    {
        $response = $this->postJson('/api/v1/auth/register', [
            'name' => 'Test User',
            'email' => 'test@example.com',
            'password' => 'password',
            'password_confirmation' => 'password',
        ]);

        $response->assertStatus(201)
                 ->assertJsonStructure([
                     'user' => ['id', 'name', 'email'],
                     'token',
                     'token_type',
                 ])
                 ->assertJson([
                     'user' => [
                         'name' => 'Test User',
                         'email' => 'test@example.com',
                     ],
                 ]);

        // ตรวจสอบ database
        $this->assertDatabaseHas('users', [
            'email' => 'test@example.com',
        ]);
    }

    /** @test */
    public function registration_requires_valid_email(): void
    {
        $response = $this->postJson('/api/v1/auth/register', [
            'name' => 'Test',
            'email' => 'not-valid-email',
            'password' => 'password',
            'password_confirmation' => 'password',
        ]);

        $response->assertStatus(422)
                 ->assertJsonValidationErrors(['email']);
    }

    /** @test */
    public function user_can_login_with_valid_credentials(): void
    {
        $user = User::factory()->create([
            'email' => 'test@example.com',
            'password' => bcrypt('password'),
        ]);

        $response = $this->postJson('/api/v1/auth/login', [
            'email' => 'test@example.com',
            'password' => 'password',
        ]);

        $response->assertStatus(200)
                 ->assertJsonStructure(['token']);
    }

    /** @test */
    public function user_cannot_login_with_wrong_password(): void
    {
        $user = User::factory()->create();

        $response = $this->postJson('/api/v1/auth/login', [
            'email' => $user->email,
            'password' => 'wrong_password',
        ]);

        $response->assertStatus(422)
                 ->assertJsonValidationErrors(['email']);
    }

    /** @test */
    public function authenticated_user_can_logout(): void
    {
        $user = User::factory()->create();
        
        // login ก่อน
        $token = $user->createToken('test')->plainTextToken;

        $response = $this->withToken($token)
                         ->postJson('/api/v1/auth/logout');

        $response->assertStatus(200);
        
        // ตรวจสอบว่า token ถูกลบแล้ว
        $this->assertDatabaseCount('personal_access_tokens', 0);
    }
}
```

---

## 4. Todo API Tests

```php
<?php
// tests/Feature/Api/TodoTest.php

namespace Tests\Feature\Api;

use App\Models\Todo;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class TodoTest extends TestCase
{
    use RefreshDatabase;

    private User $user;
    private string $token;

    protected function setUp(): void
    {
        parent::setUp();
        
        // สร้าง user และ token สำหรับทุก test
        $this->user = User::factory()->create();
        $this->token = $this->user->createToken('test')->plainTextToken;
    }

    // Helper method
    private function actingAsUser(): static
    {
        return $this->withToken($this->token);
    }

    /** @test */
    public function authenticated_user_can_get_their_todos(): void
    {
        // สร้าง todos ของ user
        Todo::factory()->count(3)->create(['user_id' => $this->user->id]);
        
        // สร้าง todos ของ user อื่น (ไม่ควรปรากฏ)
        $otherUser = User::factory()->create();
        Todo::factory()->count(2)->create(['user_id' => $otherUser->id]);

        $response = $this->actingAsUser()
                         ->getJson('/api/v1/todos');

        $response->assertStatus(200)
                 ->assertJsonCount(3, 'data')
                 ->assertJsonStructure([
                     'data' => [
                         '*' => ['id', 'title', 'status', 'priority'],
                     ],
                     'meta' => ['total', 'per_page'],
                 ]);
    }

    /** @test */
    public function unauthenticated_user_cannot_access_todos(): void
    {
        $response = $this->getJson('/api/v1/todos');

        $response->assertStatus(401);
    }

    /** @test */
    public function user_can_create_a_todo(): void
    {
        $response = $this->actingAsUser()
                         ->postJson('/api/v1/todos', [
                             'title' => 'Learn Testing',
                             'priority' => 'high',
                             'due_date' => now()->addDays(7)->format('Y-m-d'),
                         ]);

        $response->assertStatus(201)
                 ->assertJson([
                     'todo' => [
                         'title' => 'Learn Testing',
                         'priority' => 'high',
                         'status' => 'pending',
                     ],
                 ]);

        $this->assertDatabaseHas('todos', [
            'title' => 'Learn Testing',
            'user_id' => $this->user->id,
        ]);
    }

    /** @test */
    public function creating_todo_requires_title(): void
    {
        $response = $this->actingAsUser()
                         ->postJson('/api/v1/todos', [
                             'priority' => 'high',
                         ]);

        $response->assertStatus(422)
                 ->assertJsonValidationErrors(['title']);
    }

    /** @test */
    public function user_can_view_their_own_todo(): void
    {
        $todo = Todo::factory()->create(['user_id' => $this->user->id]);

        $response = $this->actingAsUser()
                         ->getJson("/api/v1/todos/{$todo->id}");

        $response->assertStatus(200)
                 ->assertJson([
                     'todo' => ['id' => $todo->id],
                 ]);
    }

    /** @test */
    public function user_cannot_view_other_users_todo(): void
    {
        $otherUser = User::factory()->create();
        $todo = Todo::factory()->create(['user_id' => $otherUser->id]);

        $response = $this->actingAsUser()
                         ->getJson("/api/v1/todos/{$todo->id}");

        $response->assertStatus(403);
    }

    /** @test */
    public function user_can_update_their_todo(): void
    {
        $todo = Todo::factory()->create([
            'user_id' => $this->user->id,
            'title' => 'Old Title',
        ]);

        $response = $this->actingAsUser()
                         ->patchJson("/api/v1/todos/{$todo->id}", [
                             'title' => 'New Title',
                             'status' => 'in_progress',
                         ]);

        $response->assertStatus(200)
                 ->assertJson([
                     'todo' => ['title' => 'New Title'],
                 ]);

        $this->assertDatabaseHas('todos', [
            'id' => $todo->id,
            'title' => 'New Title',
            'status' => 'in_progress',
        ]);
    }

    /** @test */
    public function user_can_delete_their_todo(): void
    {
        $todo = Todo::factory()->create(['user_id' => $this->user->id]);

        $response = $this->actingAsUser()
                         ->deleteJson("/api/v1/todos/{$todo->id}");

        $response->assertStatus(204);
        
        $this->assertSoftDeleted('todos', ['id' => $todo->id]);
    }

    /** @test */
    public function user_can_complete_a_todo(): void
    {
        $todo = Todo::factory()->create([
            'user_id' => $this->user->id,
            'status' => 'pending',
        ]);

        $response = $this->actingAsUser()
                         ->patchJson("/api/v1/todos/{$todo->id}/complete");

        $response->assertStatus(200);

        $this->assertDatabaseHas('todos', [
            'id' => $todo->id,
            'status' => 'completed',
        ]);
        
        $this->assertNotNull($todo->fresh()->completed_at);
    }

    /** @test */
    public function todos_can_be_filtered_by_status(): void
    {
        Todo::factory()->count(3)->create([
            'user_id' => $this->user->id,
            'status' => 'pending',
        ]);
        Todo::factory()->count(2)->create([
            'user_id' => $this->user->id,
            'status' => 'completed',
        ]);

        $response = $this->actingAsUser()
                         ->getJson('/api/v1/todos?status=pending');

        $response->assertStatus(200)
                 ->assertJsonCount(3, 'data');
    }
}
```

---

## 5. Database Testing

```php
// RefreshDatabase - reset ทุก test (ช้ากว่า แต่ clean)
use Illuminate\Foundation\Testing\RefreshDatabase;

// DatabaseTransactions - rollback หลังแต่ละ test (เร็วกว่า)
use Illuminate\Foundation\Testing\DatabaseTransactions;

// DatabaseMigrations - รัน migration ใหม่ทุกครั้ง
use Illuminate\Foundation\Testing\DatabaseMigrations;
```

```php
// Assertions
$this->assertDatabaseHas('todos', ['title' => 'Test']);
$this->assertDatabaseMissing('todos', ['title' => 'Deleted Todo']);
$this->assertDatabaseCount('todos', 5);
$this->assertSoftDeleted('todos', ['id' => 1]);
$this->assertNotSoftDeleted('todos', ['id' => 2]);

// Model assertions
$this->assertModelExists($todo);
$this->assertModelMissing($deletedTodo);
```

---

## 6. Factories

```php
<?php
// database/factories/TodoFactory.php

namespace Database\Factories;

use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;

class TodoFactory extends Factory
{
    public function definition(): array
    {
        $status = fake()->randomElement(['pending', 'in_progress', 'completed', 'cancelled']);
        
        return [
            'user_id' => User::factory(),
            'title' => fake()->sentence(4),
            'description' => fake()->optional()->paragraph(),
            'status' => $status,
            'priority' => fake()->randomElement(['low', 'medium', 'high']),
            'due_date' => fake()->optional()->dateTimeBetween('now', '+30 days'),
            'completed_at' => $status === 'completed' ? fake()->dateTimeBetween('-30 days', 'now') : null,
        ];
    }

    // States
    public function pending(): static
    {
        return $this->state(['status' => 'pending', 'completed_at' => null]);
    }

    public function completed(): static
    {
        return $this->state([
            'status' => 'completed',
            'completed_at' => now(),
        ]);
    }

    public function highPriority(): static
    {
        return $this->state(['priority' => 'high']);
    }

    public function overdue(): static
    {
        return $this->state([
            'due_date' => fake()->dateTimeBetween('-30 days', '-1 day'),
            'status' => 'pending',
        ]);
    }
}
```

```php
// ใช้ Factory ใน tests
$todo = Todo::factory()->create();
$todo = Todo::factory()->pending()->create();
$todo = Todo::factory()->completed()->highPriority()->create();

// สร้างหลายรายการ
$todos = Todo::factory()->count(5)->create(['user_id' => $user->id]);

// ไม่บันทึก database
$todo = Todo::factory()->make();
```

---

## 7. Mocking

### 7.1 Mock Services

```php
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Facades\Queue;

/** @test */
public function it_sends_welcome_email_on_registration(): void
{
    Mail::fake(); // ไม่ส่งจริง

    $response = $this->postJson('/api/v1/auth/register', [
        'name' => 'Test',
        'email' => 'test@example.com',
        'password' => 'password',
        'password_confirmation' => 'password',
    ]);

    $response->assertStatus(201);

    // ตรวจสอบว่า email ถูกส่ง
    Mail::assertSent(\App\Mail\WelcomeEmail::class, function ($mail) {
        return $mail->hasTo('test@example.com');
    });

    // ตรวจสอบจำนวน
    Mail::assertSentCount(1);
}

/** @test */
public function it_queues_jobs_on_order(): void
{
    Queue::fake();

    // ดำเนินการที่ควร dispatch job
    $this->actingAs($user)
         ->postJson('/api/orders', $orderData);

    Queue::assertPushed(\App\Jobs\ProcessPayment::class);
    Queue::assertPushedOn('payments', \App\Jobs\ProcessPayment::class);
}
```

### 7.2 Mock External Services

```php
use App\Services\PaymentGateway;
use Mockery;

/** @test */
public function it_processes_payment_successfully(): void
{
    // Mock payment gateway
    $mockGateway = Mockery::mock(PaymentGateway::class);
    $mockGateway->shouldReceive('charge')
                ->once()
                ->with(1000, 'USD')
                ->andReturn([
                    'success' => true,
                    'transaction_id' => 'txn_123',
                ]);

    // Bind mock ใน container
    $this->app->instance(PaymentGateway::class, $mockGateway);

    $response = $this->actingAsUser()
                     ->postJson('/api/orders/pay', [
                         'order_id' => $this->order->id,
                         'amount' => 1000,
                     ]);

    $response->assertStatus(200)
             ->assertJson(['transaction_id' => 'txn_123']);
}
```

### 7.3 HTTP Fake

```php
use Illuminate\Support\Facades\Http;

/** @test */
public function it_syncs_with_external_api(): void
{
    Http::fake([
        'api.external.com/products*' => Http::response([
            'products' => [
                ['id' => 1, 'name' => 'Product A', 'price' => 100],
            ],
        ], 200),
        
        'api.stripe.com/*' => Http::response(['id' => 'pi_123'], 200),
        
        '*' => Http::response('Server Error', 500),
    ]);

    $response = $this->actingAsUser()
                     ->postJson('/api/sync-products');

    $response->assertStatus(200);

    Http::assertSent(function ($request) {
        return $request->url() === 'https://api.external.com/products' &&
               $request->method() === 'GET';
    });
}
```

---

## 8. Workshop: Test สำหรับ Blog API

### 8.1 Post Tests

```php
<?php
// tests/Feature/Api/PostTest.php

namespace Tests\Feature\Api;

use App\Models\Post;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;
use Tests\TestCase;

class PostTest extends TestCase
{
    use RefreshDatabase;

    private User $author;

    protected function setUp(): void
    {
        parent::setUp();
        $this->author = User::factory()->create(['role' => 'author']);
    }

    /** @test */
    public function anyone_can_view_published_posts(): void
    {
        Post::factory()->count(3)->published()->create();
        Post::factory()->count(2)->draft()->create(); // ไม่ควรปรากฏ

        $response = $this->getJson('/api/v1/posts');

        $response->assertStatus(200)
                 ->assertJsonCount(3, 'data');
    }

    /** @test */
    public function author_can_create_post_with_image(): void
    {
        Storage::fake('public');
        
        $image = UploadedFile::fake()->image('post-image.jpg', 800, 600);

        $response = $this->actingAs($this->author, 'sanctum')
                         ->postJson('/api/v1/posts', [
                             'title' => 'Test Post',
                             'content' => 'Post content here...',
                             'status' => 'draft',
                             'image' => $image,
                         ]);

        $response->assertStatus(201)
                 ->assertJsonStructure([
                     'post' => ['id', 'title', 'image_url'],
                 ]);

        // ตรวจสอบว่าไฟล์ถูกบันทึก
        Storage::disk('public')->assertExists(
            'posts/' . date('Y/m') . '/' . $image->hashName()
        );
    }

    /** @test */
    public function author_cannot_create_post_with_huge_image(): void
    {
        $hugeImage = UploadedFile::fake()->image('big.jpg')->size(6000); // 6MB

        $response = $this->actingAs($this->author, 'sanctum')
                         ->postJson('/api/v1/posts', [
                             'title' => 'Test',
                             'content' => 'Content',
                             'image' => $hugeImage,
                         ]);

        $response->assertStatus(422)
                 ->assertJsonValidationErrors(['image']);
    }

    /** @test */
    public function author_can_publish_their_draft(): void
    {
        $post = Post::factory()->draft()->create([
            'user_id' => $this->author->id,
        ]);

        $response = $this->actingAs($this->author, 'sanctum')
                         ->patchJson("/api/v1/posts/{$post->id}/publish");

        $response->assertStatus(200);

        $this->assertDatabaseHas('posts', [
            'id' => $post->id,
            'status' => 'published',
        ]);
        
        $this->assertNotNull($post->fresh()->published_at);
    }

    /** @test */
    public function author_cannot_publish_others_post(): void
    {
        $otherAuthor = User::factory()->create(['role' => 'author']);
        $post = Post::factory()->draft()->create(['user_id' => $otherAuthor->id]);

        $response = $this->actingAs($this->author, 'sanctum')
                         ->patchJson("/api/v1/posts/{$post->id}/publish");

        $response->assertStatus(403);
    }

    /** @test */
    public function posts_can_be_searched(): void
    {
        Post::factory()->published()->create(['title' => 'Laravel Tutorial']);
        Post::factory()->published()->create(['title' => 'Vue.js Guide']);
        Post::factory()->published()->create(['title' => 'PHP Basics']);

        $response = $this->getJson('/api/v1/posts?search=Laravel');

        $response->assertStatus(200)
                 ->assertJsonCount(1, 'data')
                 ->assertJsonPath('data.0.title', 'Laravel Tutorial');
    }

    /**
     * @test
     * @dataProvider postValidationProvider
     */
    public function post_creation_validates_required_fields(
        array $data,
        array $invalidFields
    ): void {
        $response = $this->actingAs($this->author, 'sanctum')
                         ->postJson('/api/v1/posts', $data);

        $response->assertStatus(422)
                 ->assertJsonValidationErrors($invalidFields);
    }

    public static function postValidationProvider(): array
    {
        return [
            'missing title' => [
                ['content' => 'Some content'],
                ['title'],
            ],
            'missing content' => [
                ['title' => 'Test Title'],
                ['content'],
            ],
            'title too long' => [
                ['title' => str_repeat('a', 256), 'content' => 'Content'],
                ['title'],
            ],
        ];
    }
}
```

### 8.2 Comment Tests

```php
<?php
// tests/Feature/Api/CommentTest.php

namespace Tests\Feature\Api;

use App\Models\Comment;
use App\Models\Post;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class CommentTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function authenticated_user_can_comment_on_published_post(): void
    {
        $user = User::factory()->create();
        $post = Post::factory()->published()->create();

        $response = $this->actingAs($user, 'sanctum')
                         ->postJson("/api/v1/posts/{$post->id}/comments", [
                             'content' => 'Great post!',
                         ]);

        $response->assertStatus(201);
        
        $this->assertDatabaseHas('comments', [
            'post_id' => $post->id,
            'user_id' => $user->id,
            'content' => 'Great post!',
        ]);
    }

    /** @test */
    public function user_cannot_comment_on_draft_post(): void
    {
        $user = User::factory()->create();
        $post = Post::factory()->draft()->create();

        $response = $this->actingAs($user, 'sanctum')
                         ->postJson("/api/v1/posts/{$post->id}/comments", [
                             'content' => 'Comment on draft',
                         ]);

        $response->assertStatus(403);
    }

    /** @test */
    public function user_can_delete_their_own_comment(): void
    {
        $user = User::factory()->create();
        $comment = Comment::factory()->create(['user_id' => $user->id]);

        $response = $this->actingAs($user, 'sanctum')
                         ->deleteJson("/api/v1/comments/{$comment->id}");

        $response->assertStatus(204);
        $this->assertModelMissing($comment);
    }
}
```

---

## 9. Test Helpers

```php
<?php
// tests/TestCase.php

namespace Tests;

use App\Models\User;
use Illuminate\Foundation\Testing\TestCase as BaseTestCase;
use Laravel\Sanctum\Sanctum;

abstract class TestCase extends BaseTestCase
{
    use CreatesApplication;

    // Helper: login as user
    protected function loginAsUser(?User $user = null): User
    {
        $user ??= User::factory()->create();
        Sanctum::actingAs($user);
        return $user;
    }

    // Helper: login as admin
    protected function loginAsAdmin(): User
    {
        $admin = User::factory()->admin()->create();
        Sanctum::actingAs($admin);
        return $admin;
    }

    // Helper: ส่ง API request พร้อม token
    protected function apiAs(User $user, string $method, string $uri, array $data = []): \Illuminate\Testing\TestResponse
    {
        $token = $user->createToken('test')->plainTextToken;
        
        return $this->withToken($token)
                    ->{$method . 'Json'}($uri, $data);
    }
}
```

---

## Quiz

### คำถาม 1
`RefreshDatabase` vs `DatabaseTransactions` ต่างกันอย่างไร?

**A)** ไม่ต่างกัน  
**B)** RefreshDatabase รัน migration ใหม่ทุก test, DatabaseTransactions rollback transaction หลัง test  
**C)** RefreshDatabase เร็วกว่า  
**D)** DatabaseTransactions ใช้ได้เฉพาะกับ MySQL  

**เฉลย: B** - `RefreshDatabase` clean database โดย rollback/remigrate, `DatabaseTransactions` wrap ใน transaction แล้ว rollback (เร็วกว่า แต่ไม่รองรับ transactions ใน code ที่ test)

---

### คำถาม 2
`Mail::fake()` ทำอะไร?

**A)** สร้าง test email address  
**B)** ป้องกันการส่ง email จริง และให้สามารถ assert การส่ง email ได้  
**C)** บันทึก email ลง database  
**D)** ส่ง email ไปยัง mailtrap  

**เฉลย: B** - `Mail::fake()` swap mailer ด้วย fake implementation ที่บันทึก mails แทนการส่งจริง

---

### คำถาม 3
`assertSoftDeleted('todos', ['id' => 1])` ตรวจสอบอะไร?

**A)** Record ถูกลบออกจาก database  
**B)** Record มี deleted_at ไม่เป็น null (soft deleted)  
**C)** Record ถูกซ่อน  
**D)** Record อยู่ใน trash table  

**เฉลย: B** - ตรวจสอบว่า record มี `deleted_at` ที่ไม่เป็น null แต่ยังอยู่ใน database

---

### คำถาม 4
`@dataProvider` annotation ใน PHPUnit ใช้ทำอะไร?

**A)** กำหนด database seeder  
**B)** รัน test หลายครั้งด้วยข้อมูลต่างกัน  
**C)** inject dependencies  
**D)** skip test  

**เฉลย: B** - `@dataProvider` ทำให้ test method รันซ้ำหลายครั้ง โดยแต่ละครั้งใช้ data ชุดที่ต่างกันจาก method ที่ระบุ

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- ✅ ตั้งค่า Testing environment (phpunit.xml, .env.testing)
- ✅ Unit Tests สำหรับ Services และ Models
- ✅ Feature Tests สำหรับ HTTP endpoints
- ✅ Database Testing assertions
- ✅ Factories สำหรับ test data
- ✅ Mocking (Mail, Queue, HTTP, Services)
- ✅ Workshop: Test suite ครบถ้วนสำหรับ Blog API

---

## ขั้นตอนต่อไป

ยินดีด้วย! คุณได้เรียนจบ Part 036-042 ของ Laravel Advanced Course แล้ว

### สรุปสิ่งที่เรียนใน Parts นี้:
- **Part 036:** Artisan Commands & Scheduling
- **Part 037:** Queues & Background Jobs
- **Part 038:** Events, Listeners & Broadcasting
- **Part 039:** Mail & Notifications
- **Part 040:** File Storage & Image Processing
- **Part 041:** REST API Development
- **Part 042:** Testing

### แนะนำ Topics ต่อไป:
- Laravel Octane (Performance)
- Laravel Scout (Full-text search)
- Laravel Telescope (Debug & monitoring)
- Laravel Horizon (Queue monitoring)
- Docker & Deployment
- CI/CD Pipeline
