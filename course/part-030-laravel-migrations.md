# Part 030: Laravel Migrations, Seeders & Factories

**ระดับ: กลาง-สูง (Intermediate-Advanced)**

---

## เป้าหมายของ Part นี้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- สร้างและรัน Migration อย่างถูกต้อง
- ใช้งาน Column types ทุกประเภท
- กำหนด Foreign Keys และ Indexes
- ทำ Rollback, Reset และ Fresh migrations
- สร้าง Seeders เพื่อเติมข้อมูลเริ่มต้น
- สร้าง Factories สำหรับ fake data
- สร้าง Database schema สมบูรณ์สำหรับ E-commerce

---

## 1. สร้าง Migration

### สร้างด้วย Artisan

```bash
# สร้าง table ใหม่
php artisan make:migration create_posts_table

# เพิ่ม column ใน table ที่มีอยู่
php artisan make:migration add_published_at_to_posts_table --table=posts

# ลบ column
php artisan make:migration remove_deprecated_column_from_posts_table --table=posts

# Rename table
php artisan make:migration rename_posts_to_articles

# สร้าง Model พร้อม Migration
php artisan make:model Post -m
php artisan make:model Post --migration

# สร้างครบ
php artisan make:model Post -mfsc
# -m = migration, -f = factory, -s = seeder, -c = controller
```

### โครงสร้าง Migration File

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * ชื่อ connection (null = default connection)
     */
    protected $connection = null; // หรือ 'mysql', 'pgsql', etc.

    /**
     * run migration
     */
    public function up(): void
    {
        Schema::create('posts', function (Blueprint $table) {
            // Column definitions
            $table->id();
            $table->string('title');
            $table->timestamps();
        });
    }

    /**
     * undo migration
     */
    public function down(): void
    {
        Schema::dropIfExists('posts');
    }
};
```

---

## 2. Column Types ทั้งหมด

### Integer Types

```php
Schema::create('example', function (Blueprint $table) {
    // Auto-increment IDs
    $table->id();                    // BIGINT UNSIGNED, PRIMARY KEY, AUTO_INCREMENT (alias ของ bigIncrements)
    $table->increments('id');        // INT UNSIGNED, AUTO_INCREMENT
    $table->bigIncrements('id');     // BIGINT UNSIGNED, AUTO_INCREMENT
    $table->smallIncrements('id');   // SMALLINT UNSIGNED, AUTO_INCREMENT
    $table->tinyIncrements('id');    // TINYINT UNSIGNED, AUTO_INCREMENT
    $table->mediumIncrements('id');  // MEDIUMINT UNSIGNED, AUTO_INCREMENT

    // Integer columns (ไม่ auto-increment)
    $table->integer('votes');              // INT
    $table->bigInteger('population');      // BIGINT
    $table->smallInteger('level');         // SMALLINT
    $table->tinyInteger('is_active');      // TINYINT
    $table->mediumInteger('views');        // MEDIUMINT

    // Unsigned integers
    $table->unsignedInteger('score');          // INT UNSIGNED
    $table->unsignedBigInteger('file_size');   // BIGINT UNSIGNED
    $table->unsignedSmallInteger('age');       // SMALLINT UNSIGNED
    $table->unsignedTinyInteger('rating');     // TINYINT UNSIGNED (0-255)
    $table->unsignedMediumInteger('priority'); // MEDIUMINT UNSIGNED
});
```

### String & Text Types

```php
Schema::create('example', function (Blueprint $table) {
    // String (มี max length)
    $table->string('name');           // VARCHAR(255)
    $table->string('code', 10);       // VARCHAR(10)
    $table->char('country', 2);       // CHAR(2)

    // Text (ไม่มี limit)
    $table->text('content');          // TEXT (~65KB)
    $table->mediumText('body');       // MEDIUMTEXT (~16MB)
    $table->longText('document');     // LONGTEXT (~4GB)
    $table->tinyText('summary');      // TINYTEXT (~255 bytes)

    // TinyText + String variations
    $table->uuid('uuid');             // CHAR(36) - UUID format
    $table->ulid('ulid');             // CHAR(26) - ULID format
    $table->ipAddress('ip');          // VARCHAR(45) - IPv4 หรือ IPv6
    $table->macAddress('mac');        // VARCHAR(17)
    $table->rememberToken();          // VARCHAR(100) สำหรับ "remember me"
    $table->slug('slug');             // ไม่มีใน Laravel จริงๆ ใช้ string แทน
});
```

### Numeric Types

```php
Schema::create('example', function (Blueprint $table) {
    // Decimal (fixed precision - เหมาะกับเงิน)
    $table->decimal('price', 10, 2);          // DECIMAL(10,2) - 99999999.99
    $table->unsignedDecimal('amount', 8, 2);  // ไม่ติดลบ

    // Float (ความแม่นยำไม่สูง ไม่แนะนำสำหรับเงิน)
    $table->float('score', 8, 2);   // FLOAT(8,2)
    $table->double('rate', 15, 8);  // DOUBLE(15,8)

    // Boolean
    $table->boolean('is_active');             // TINYINT(1)
    $table->boolean('published')->default(false);
});
```

### Date & Time Types

```php
Schema::create('example', function (Blueprint $table) {
    // Date/Time
    $table->date('birth_date');              // DATE
    $table->time('start_time');             // TIME
    $table->dateTime('published_at');       // DATETIME
    $table->dateTimeTz('event_at');         // DATETIME with timezone
    $table->timestamp('verified_at');       // TIMESTAMP
    $table->timestampTz('expires_at');      // TIMESTAMP with timezone
    $table->year('graduation_year');        // YEAR

    // Auto timestamps
    $table->timestamps();                   // created_at และ updated_at (TIMESTAMP, nullable)
    $table->nullableTimestamps();           // เหมือน timestamps() แต่ nullable
    $table->timestampsTz();                 // Timestamps with timezone
    $table->softDeletes();                  // deleted_at (TIMESTAMP, nullable)
    $table->softDeletesTz();               // deleted_at with timezone
});
```

### JSON & Binary Types

```php
Schema::create('example', function (Blueprint $table) {
    // JSON
    $table->json('settings');       // JSON
    $table->jsonb('metadata');      // JSONB (PostgreSQL)

    // Binary
    $table->binary('data');         // BLOB
    $table->binary('hash', 32);     // BLOB(32)

    // Enum
    $table->enum('status', ['draft', 'published', 'archived']); // ENUM
    $table->set('permissions', ['read', 'write', 'admin']);       // SET (MySQL)
});
```

### Special Types

```php
Schema::create('example', function (Blueprint $table) {
    $table->foreignId('user_id');          // BIGINT UNSIGNED (สำหรับ foreign key)
    $table->foreignUlid('post_ulid');      // CHAR(26) (สำหรับ ULID foreign key)
    $table->foreignUuid('category_uuid'); // CHAR(36) (สำหรับ UUID foreign key)

    $table->morphs('taggable');
    // เพิ่ม: taggable_id (BIGINT UNSIGNED)
    //        taggable_type (VARCHAR(255))

    $table->nullableMorphs('commentable');
    // เพิ่ม: commentable_id (BIGINT UNSIGNED, nullable)
    //        commentable_type (VARCHAR(255), nullable)

    $table->ulidMorphs('actionable');      // ULID morphs
    $table->uuidMorphs('mediable');        // UUID morphs
});
```

### Column Modifiers

```php
Schema::create('example', function (Blueprint $table) {
    // Nullable
    $table->string('nickname')->nullable();

    // Default values
    $table->boolean('active')->default(true);
    $table->string('role')->default('user');
    $table->integer('views')->default(0);
    $table->timestamp('verified_at')->useCurrent(); // DEFAULT CURRENT_TIMESTAMP

    // Auto-update timestamp
    $table->timestamp('updated_at')->useCurrentOnUpdate();

    // Comments (MySQL)
    $table->string('email')->comment('อีเมลสำหรับ login');

    // Column order (MySQL)
    $table->string('first_name');
    $table->string('last_name')->after('first_name');
    $table->string('prefix')->first(); // วางเป็น column แรก

    // Invisible column (MySQL 8.0+)
    $table->string('internal_code')->invisible();

    // Collation
    $table->string('name')->collation('utf8mb4_unicode_ci');

    // Charset
    $table->string('content')->charset('utf8mb4');

    // Unsigned (integer only)
    $table->integer('score')->unsigned();

    // Auto increment
    $table->integer('sequence')->autoIncrement();
    
    // Stored (computed column)
    $table->string('full_name')->storedAs("CONCAT(first_name, ' ', last_name)");
    $table->string('full_name')->virtualAs("CONCAT(first_name, ' ', last_name)");
});
```

---

## 3. Foreign Keys

### กำหนด Foreign Keys

```php
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    
    // วิธีที่ 1: foreignId (แนะนำ - รวบรัด)
    $table->foreignId('user_id')->constrained();
    // สร้าง user_id BIGINT UNSIGNED + foreign key -> users(id)
    
    $table->foreignId('category_id')
        ->nullable()
        ->constrained()
        ->nullOnDelete(); // SET NULL เมื่อลบ category
    
    $table->foreignId('author_id')
        ->constrained('users') // ระบุ table ที่ต้องการถ้าชื่อไม่ตรง convention
        ->cascadeOnDelete()    // CASCADE เมื่อลบ user
        ->cascadeOnUpdate();
    
    // วิธีที่ 2: แบบ explicit
    $table->unsignedBigInteger('parent_id')->nullable();
    $table->foreign('parent_id')
        ->references('id')
        ->on('posts')
        ->nullOnDelete();
    
    $table->timestamps();
});
```

### Foreign Key Actions

```php
// ON DELETE actions
->cascadeOnDelete()       // CASCADE: ลบ child records ด้วย
->restrictOnDelete()      // RESTRICT: ป้องกันการลบถ้ามี child
->nullOnDelete()          // SET NULL: set FK เป็น null
->noActionOnDelete()      // NO ACTION: (คล้าย RESTRICT)

// ON UPDATE actions
->cascadeOnUpdate()       // CASCADE: อัปเดต FK ใน child ด้วย
->restrictOnUpdate()      // RESTRICT: ป้องกันการอัปเดตถ้ามี child
->nullOnUpdate()          // SET NULL: set FK เป็น null
->noActionOnUpdate()      // NO ACTION

// Naming convention
// ชื่อ foreign key constraint จะเป็น: posts_user_id_foreign
// สามารถกำหนดเองได้:
$table->foreign('user_id', 'fk_posts_user')
    ->references('id')
    ->on('users');
```

### ลบ Foreign Key

```php
public function down(): void
{
    Schema::table('posts', function (Blueprint $table) {
        // ลบ foreign key ก่อนลบ column
        $table->dropForeign(['user_id']);
        // หรือ
        $table->dropForeign('posts_user_id_foreign');
        
        $table->dropColumn('user_id');
    });
}
```

---

## 4. Indexes

### ประเภท Index

```php
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->string('slug');
    $table->string('email');
    $table->text('content');
    $table->string('category');
    $table->enum('status', ['draft', 'published']);
    $table->timestamp('published_at')->nullable();
    $table->timestamps();
    
    // Primary Key (ทำโดย id() อัตโนมัติ)
    $table->primary('id'); // ถ้ากำหนดเอง

    // Unique Index (ค่าไม่ซ้ำ)
    $table->unique('slug');           // index ชื่อ posts_slug_unique
    $table->unique('email');
    $table->unique(['email', 'type']); // Composite unique

    // Regular Index (เร็วขึ้นในการ query)
    $table->index('status');
    $table->index(['status', 'published_at']); // Composite index
    $table->index('category', 'idx_posts_category'); // กำหนดชื่อเอง

    // Full-text Search Index
    $table->fullText(['title', 'content']);  // สำหรับ MySQL/PostgreSQL

    // Spatial Index (สำหรับ Geographic data)
    $table->point('location')->nullable();
    $table->spatialIndex('location');
});
```

### แก้ไข Indexes

```php
Schema::table('posts', function (Blueprint $table) {
    // เพิ่ม index
    $table->index('views');
    $table->unique('slug');

    // ลบ index
    $table->dropIndex(['views']);                  // ลบจาก column name
    $table->dropIndex('posts_views_index');        // ลบจาก index name
    $table->dropUnique(['slug']);                  // ลบ unique
    $table->dropUnique('posts_slug_unique');
    $table->dropPrimary(['id']);                   // ลบ primary key
    $table->dropFullText(['title', 'content']);    // ลบ full text
    $table->dropSpatialIndex(['location']);        // ลบ spatial
});
```

---

## 5. Schema Operations

### แก้ไข Column ที่มีอยู่

```php
// ต้องติดตั้ง doctrine/dbal ก่อน (Laravel 10 และต่ำกว่า)
// composer require doctrine/dbal
// Laravel 11+ ไม่ต้องการ

Schema::table('posts', function (Blueprint $table) {
    // เปลี่ยน type
    $table->text('content')->change();           // เปลี่ยนเป็น text
    $table->string('name', 100)->change();       // เปลี่ยน length

    // เพิ่ม/ลบ nullable
    $table->string('nickname')->nullable()->change();

    // เปลี่ยน default
    $table->boolean('active')->default(true)->change();

    // Rename column
    $table->renameColumn('title', 'headline');

    // เพิ่ม column
    $table->string('subtitle')->nullable()->after('title');

    // ลบ column
    $table->dropColumn('deprecated_field');
    $table->dropColumn(['field1', 'field2', 'field3']); // ลบหลาย columns
});
```

### Rename Table

```php
Schema::rename('posts', 'articles');
Schema::rename('users', 'members');
```

### Drop Table

```php
Schema::drop('posts');           // ลบ table (error ถ้าไม่มี)
Schema::dropIfExists('posts');   // ลบถ้ามี (safe)

// ลบหลาย tables
Schema::drop('posts');
Schema::drop('comments');
Schema::drop('tags');
```

### Check Table/Column

```php
// ตรวจสอบว่า table มีอยู่
if (Schema::hasTable('posts')) {
    // table มีอยู่
}

// ตรวจสอบว่า column มีอยู่
if (Schema::hasColumn('posts', 'published_at')) {
    // column มีอยู่
}

// ตรวจสอบหลาย columns
if (Schema::hasColumns('posts', ['title', 'content', 'slug'])) {
    // ทุก column มีอยู่
}

// ดูชนิดของ column
$type = Schema::getColumnType('posts', 'views');
// returns: 'integer'
```

---

## 6. รัน Migration

```bash
# รัน migrations ที่ยังไม่ได้รัน
php artisan migrate

# รัน พร้อม verbose output
php artisan migrate --verbose

# ดูว่า migration ไหนจะรัน (dry run)
php artisan migrate --pretend

# รัน specific migration path
php artisan migrate --path=/database/migrations/tenant

# Force run (production)
php artisan migrate --force

# ดูสถานะ migrations
php artisan migrate:status
```

---

## 7. Rollback และ Reset

```bash
# Rollback migration ล่าสุด (1 batch)
php artisan migrate:rollback

# Rollback 3 migrations ล่าสุด (3 steps)
php artisan migrate:rollback --step=3

# Rollback specific batch
php artisan migrate:rollback --batch=2

# Reset - rollback ทั้งหมด
php artisan migrate:reset

# Fresh - ลบทุก table และ migrate ใหม่ทั้งหมด
php artisan migrate:fresh

# Fresh + Seed
php artisan migrate:fresh --seed

# Fresh เฉพาะ seeder ที่กำหนด
php artisan migrate:fresh --seeder=TestSeeder

# Refresh = rollback ทั้งหมด + migrate ใหม่
php artisan migrate:refresh

# Refresh + Seed
php artisan migrate:refresh --seed

# Refresh เฉพาะ N steps ล่าสุด
php artisan migrate:refresh --step=5
```

---

## 8. Seeders

Seeders ใช้เติมข้อมูลเริ่มต้นหรือ test data

### สร้าง Seeder

```bash
php artisan make:seeder UserSeeder
php artisan make:seeder PostSeeder
php artisan make:seeder CategorySeeder
```

### Seeder พื้นฐาน

```php
// database/seeders/CategorySeeder.php
<?php

namespace Database\Seeders;

use App\Models\Category;
use Illuminate\Database\Seeder;

class CategorySeeder extends Seeder
{
    public function run(): void
    {
        $categories = [
            ['name' => 'เทคโนโลยี',  'slug' => 'technology',  'color' => '#3B82F6'],
            ['name' => 'ท่องเที่ยว', 'slug' => 'travel',      'color' => '#10B981'],
            ['name' => 'อาหาร',      'slug' => 'food',        'color' => '#F59E0B'],
            ['name' => 'สุขภาพ',     'slug' => 'health',      'color' => '#EF4444'],
            ['name' => 'ธุรกิจ',     'slug' => 'business',    'color' => '#8B5CF6'],
            ['name' => 'ไลฟ์สไตล์', 'slug' => 'lifestyle',   'color' => '#EC4899'],
        ];

        foreach ($categories as $category) {
            Category::firstOrCreate(
                ['slug' => $category['slug']],
                $category
            );
        }

        $this->command->info('✓ สร้าง categories เรียบร้อย');
    }
}
```

### DatabaseSeeder (Main Seeder)

```php
// database/seeders/DatabaseSeeder.php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        // รัน seeders ตามลำดับที่กำหนด
        $this->call([
            CategorySeeder::class,
            TagSeeder::class,
            UserSeeder::class,
            PostSeeder::class,
            CommentSeeder::class,
        ]);

        $this->command->info('✓ Database seeding เสร็จสิ้น!');
    }
}
```

### รัน Seeder

```bash
# รัน DatabaseSeeder
php artisan db:seed

# รัน specific seeder
php artisan db:seed --class=PostSeeder

# รัน fresh + seed
php artisan migrate:fresh --seed
```

---

## 9. Factories

Factories ใช้สร้าง fake data สำหรับ testing และ development

### สร้าง Factory

```bash
php artisan make:factory PostFactory --model=Post
php artisan make:factory UserFactory --model=User
```

### User Factory

```php
// database/factories/UserFactory.php
<?php

namespace Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Str;

class UserFactory extends Factory
{
    public function definition(): array
    {
        return [
            'name'              => fake()->name(),
            'email'             => fake()->unique()->safeEmail(),
            'email_verified_at' => now(),
            'password'          => bcrypt('password'), // ใช้ 'password' ทุกคน
            'remember_token'    => Str::random(10),
            'role'              => 'user',
            'bio'               => fake()->paragraph(2),
            'avatar'            => 'avatars/default.jpg',
            'created_at'        => fake()->dateTimeBetween('-2 years', 'now'),
        ];
    }

    /**
     * Admin state
     */
    public function admin(): static
    {
        return $this->state(fn (array $attributes) => [
            'role' => 'admin',
        ]);
    }

    /**
     * Unverified state
     */
    public function unverified(): static
    {
        return $this->state(fn (array $attributes) => [
            'email_verified_at' => null,
        ]);
    }

    /**
     * Thai user state
     */
    public function thai(): static
    {
        return $this->state(fn (array $attributes) => [
            'name'  => fake('th_TH')->name(),
            'email' => fake()->unique()->safeEmail(),
        ]);
    }
}
```

### Post Factory

```php
// database/factories/PostFactory.php
<?php

namespace Database\Factories;

use App\Models\User;
use App\Models\Category;
use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Str;

class PostFactory extends Factory
{
    public function definition(): array
    {
        $title = fake()->sentence(fake()->numberBetween(4, 10));
        $title = rtrim($title, '.'); // ลบ period ท้าย sentence

        return [
            'user_id'      => User::factory(),     // สร้าง User ใหม่ถ้าไม่มี
            'category_id'  => Category::inRandomOrder()->first()?->id,
            'title'        => $title,
            'slug'         => Str::slug($title),
            'excerpt'      => fake()->paragraph(2),
            'content'      => implode("\n\n", fake()->paragraphs(fake()->numberBetween(5, 15))),
            'image'        => null,
            'status'       => fake()->randomElement(['draft', 'published']),
            'views'        => fake()->numberBetween(0, 10000),
            'published_at' => fake()->optional(0.8)->dateTimeBetween('-1 year', 'now'),
            'created_at'   => fake()->dateTimeBetween('-2 years', 'now'),
        ];
    }

    /**
     * Published state
     */
    public function published(): static
    {
        return $this->state(fn (array $attributes) => [
            'status'       => 'published',
            'published_at' => fake()->dateTimeBetween('-1 year', 'now'),
        ]);
    }

    /**
     * Draft state
     */
    public function draft(): static
    {
        return $this->state(fn (array $attributes) => [
            'status'       => 'draft',
            'published_at' => null,
        ]);
    }

    /**
     * Popular state (views สูง)
     */
    public function popular(): static
    {
        return $this->state(fn (array $attributes) => [
            'views'  => fake()->numberBetween(5000, 100000),
            'status' => 'published',
        ]);
    }

    /**
     * Configure - รัน callback หลัง create
     */
    public function configure(): static
    {
        return $this->afterCreating(function (\App\Models\Post $post) {
            // เพิ่ม tags หลัง create
            $tags = \App\Models\Tag::inRandomOrder()->limit(3)->get();
            $post->tags()->attach($tags);
        });
    }
}
```

### ใช้ Factory ใน Seeder

```php
// database/seeders/PostSeeder.php
<?php

namespace Database\Seeders;

use App\Models\User;
use App\Models\Post;
use Illuminate\Database\Seeder;

class PostSeeder extends Seeder
{
    public function run(): void
    {
        // สร้าง admin user
        $admin = User::factory()->admin()->create([
            'name'  => 'Admin User',
            'email' => 'admin@example.com',
        ]);

        // สร้าง regular users
        $users = User::factory(10)->create();

        // สร้าง published posts สำหรับ admin
        Post::factory(20)
            ->published()
            ->for($admin)
            ->create();

        // สร้าง posts สำหรับแต่ละ user
        $users->each(function (User $user) {
            // Published posts
            Post::factory(fake()->numberBetween(3, 10))
                ->published()
                ->for($user)
                ->create();

            // Draft posts
            Post::factory(fake()->numberBetween(1, 3))
                ->draft()
                ->for($user)
                ->create();
        });

        // Popular posts
        Post::factory(5)
            ->popular()
            ->for($admin)
            ->create();

        $count = Post::count();
        $this->command->info("✓ สร้าง {$count} posts เรียบร้อย");
    }
}
```

### ใช้ Factory ใน Testing

```php
// tests/Feature/PostTest.php
<?php

namespace Tests\Feature;

use App\Models\Post;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class PostTest extends TestCase
{
    use RefreshDatabase; // Reset database ทุก test

    public function test_user_can_create_post(): void
    {
        // สร้าง user
        $user = User::factory()->create();

        // Act as user
        $response = $this->actingAs($user)->post('/posts', [
            'title'   => 'Test Post',
            'content' => str_repeat('content ', 20),
            'status'  => 'draft',
        ]);

        // Assert
        $response->assertRedirect();
        $this->assertDatabaseHas('posts', [
            'title'   => 'Test Post',
            'user_id' => $user->id,
        ]);
    }

    public function test_published_posts_appear_in_index(): void
    {
        Post::factory(5)->published()->create();
        Post::factory(3)->draft()->create();

        $response = $this->get('/posts');

        $response->assertStatus(200);
        $response->assertViewHas('posts');
        
        // ควรเห็นแค่ 5 published posts
        $this->assertEquals(5, $response->viewData('posts')->total());
    }
}
```

---

## Workshop: สร้าง Database Schema สำหรับ E-commerce

### โจทย์

สร้าง Database schema สมบูรณ์สำหรับ E-commerce ที่มี:
- Users และ Profiles
- Products, Categories, Variants
- Orders และ Order Items
- Reviews และ Ratings
- Cart และ Wishlist

### Schema Design

```
users
  - id, name, email, password, role, ...

user_profiles
  - id, user_id, phone, address, birth_date, ...

categories
  - id, parent_id, name, slug, image, order, ...

products
  - id, category_id, name, slug, description, base_price, stock, status, ...

product_images
  - id, product_id, path, alt, is_primary, order, ...

product_variants
  - id, product_id, sku, attributes(json), price, stock, ...

orders
  - id, user_id, status, total, shipping_fee, payment_method, ...

order_items
  - id, order_id, product_id, variant_id, quantity, price, ...

reviews
  - id, user_id, product_id, rating, comment, ...

cart_items
  - id, user_id, product_id, variant_id, quantity, ...

wishlists
  - id, user_id, product_id, ...
```

### Step 1: Users Migration

```php
// database/migrations/0001_01_01_000000_create_users_table.php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->timestamp('email_verified_at')->nullable();
            $table->string('password');
            $table->enum('role', ['customer', 'seller', 'admin'])->default('customer');
            $table->boolean('is_active')->default(true);
            $table->rememberToken();
            $table->timestamps();
            $table->softDeletes();

            $table->index(['email', 'is_active']);
        });

        Schema::create('user_profiles', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->unique()->constrained()->cascadeOnDelete();
            $table->string('phone', 20)->nullable();
            $table->text('address')->nullable();
            $table->string('city', 100)->nullable();
            $table->string('province', 100)->nullable();
            $table->string('postal_code', 10)->nullable();
            $table->date('birth_date')->nullable();
            $table->enum('gender', ['male', 'female', 'other'])->nullable();
            $table->string('avatar')->nullable();
            $table->timestamps();
        });

        Schema::create('password_reset_tokens', function (Blueprint $table) {
            $table->string('email')->primary();
            $table->string('token');
            $table->timestamp('created_at')->nullable();
        });

        Schema::create('sessions', function (Blueprint $table) {
            $table->string('id')->primary();
            $table->foreignId('user_id')->nullable()->index();
            $table->string('ip_address', 45)->nullable();
            $table->text('user_agent')->nullable();
            $table->longText('payload');
            $table->integer('last_activity')->index();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('sessions');
        Schema::dropIfExists('password_reset_tokens');
        Schema::dropIfExists('user_profiles');
        Schema::dropIfExists('users');
    }
};
```

### Step 2: Categories Migration

```php
// database/migrations/xxxx_create_categories_table.php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('categories', function (Blueprint $table) {
            $table->id();
            $table->foreignId('parent_id')
                ->nullable()
                ->constrained('categories')
                ->nullOnDelete();
            $table->string('name');
            $table->string('slug')->unique();
            $table->text('description')->nullable();
            $table->string('image')->nullable();
            $table->unsignedSmallInteger('order')->default(0);
            $table->boolean('is_active')->default(true);
            $table->timestamps();

            $table->index(['parent_id', 'is_active', 'order']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('categories');
    }
};
```

### Step 3: Products Migration

```php
// database/migrations/xxxx_create_products_table.php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('products', function (Blueprint $table) {
            $table->id();
            $table->foreignId('category_id')
                ->nullable()
                ->constrained()
                ->nullOnDelete();
            $table->foreignId('user_id') // seller
                ->constrained()
                ->cascadeOnDelete();

            $table->string('name');
            $table->string('slug')->unique();
            $table->string('sku')->unique()->nullable();
            $table->text('description')->nullable();
            $table->longText('details')->nullable(); // HTML content

            $table->decimal('base_price', 10, 2);
            $table->decimal('sale_price', 10, 2)->nullable();
            $table->decimal('cost_price', 10, 2)->nullable(); // ต้นทุน

            $table->unsignedInteger('stock')->default(0);
            $table->unsignedSmallInteger('low_stock_threshold')->default(10);

            $table->decimal('weight', 8, 2)->nullable(); // กิโลกรัม
            $table->json('dimensions')->nullable(); // {length, width, height}

            $table->enum('status', ['draft', 'active', 'inactive', 'out_of_stock'])
                ->default('draft');

            $table->boolean('is_featured')->default(false);
            $table->boolean('is_digital')->default(false);
            $table->unsignedInteger('views')->default(0);

            $table->json('attributes')->nullable(); // color, size options
            $table->json('seo')->nullable(); // meta_title, meta_description

            $table->timestamps();
            $table->softDeletes();

            $table->index(['category_id', 'status', 'is_featured']);
            $table->index(['status', 'sale_price']);
            $table->fullText(['name', 'description']);
        });

        Schema::create('product_images', function (Blueprint $table) {
            $table->id();
            $table->foreignId('product_id')->constrained()->cascadeOnDelete();
            $table->string('path');
            $table->string('alt')->nullable();
            $table->boolean('is_primary')->default(false);
            $table->unsignedSmallInteger('order')->default(0);
            $table->timestamps();

            $table->index(['product_id', 'is_primary', 'order']);
        });

        Schema::create('product_variants', function (Blueprint $table) {
            $table->id();
            $table->foreignId('product_id')->constrained()->cascadeOnDelete();
            $table->string('sku')->unique();
            $table->json('attributes'); // {'color': 'red', 'size': 'XL'}
            $table->decimal('price', 10, 2);
            $table->decimal('sale_price', 10, 2)->nullable();
            $table->unsignedInteger('stock')->default(0);
            $table->string('image')->nullable();
            $table->boolean('is_active')->default(true);
            $table->timestamps();

            $table->index(['product_id', 'is_active']);
        });

        // Product Tags (many-to-many)
        Schema::create('tags', function (Blueprint $table) {
            $table->id();
            $table->string('name')->unique();
            $table->string('slug')->unique();
            $table->timestamps();
        });

        Schema::create('product_tag', function (Blueprint $table) {
            $table->foreignId('product_id')->constrained()->cascadeOnDelete();
            $table->foreignId('tag_id')->constrained()->cascadeOnDelete();
            $table->primary(['product_id', 'tag_id']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('product_tag');
        Schema::dropIfExists('tags');
        Schema::dropIfExists('product_variants');
        Schema::dropIfExists('product_images');
        Schema::dropIfExists('products');
    }
};
```

### Step 4: Orders Migration

```php
// database/migrations/xxxx_create_orders_table.php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('orders', function (Blueprint $table) {
            $table->id();
            $table->string('order_number')->unique(); // ORD-2024-001234
            $table->foreignId('user_id')->constrained()->restrictOnDelete();

            $table->enum('status', [
                'pending',        // รอชำระเงิน
                'paid',           // ชำระแล้ว
                'processing',     // กำลังเตรียม
                'shipped',        // จัดส่งแล้ว
                'delivered',      // ส่งถึงแล้ว
                'cancelled',      // ยกเลิก
                'refunded',       // คืนเงิน
            ])->default('pending');

            // Pricing
            $table->decimal('subtotal', 10, 2);     // ราคาสินค้ารวม
            $table->decimal('discount', 10, 2)->default(0);  // ส่วนลด
            $table->decimal('shipping_fee', 10, 2)->default(0); // ค่าส่ง
            $table->decimal('tax', 10, 2)->default(0);       // ภาษี
            $table->decimal('total', 10, 2);         // รวมทั้งหมด

            // Shipping Address
            $table->string('shipping_name');
            $table->string('shipping_phone', 20);
            $table->text('shipping_address');
            $table->string('shipping_city', 100);
            $table->string('shipping_province', 100);
            $table->string('shipping_postal_code', 10);
            $table->string('shipping_country', 2)->default('TH');

            // Payment
            $table->enum('payment_method', [
                'credit_card', 'bank_transfer', 'promptpay', 'cod', 'wallet'
            ])->nullable();
            $table->string('payment_transaction_id')->nullable();
            $table->timestamp('paid_at')->nullable();

            // Shipping
            $table->string('tracking_number')->nullable();
            $table->string('shipping_provider')->nullable();
            $table->timestamp('shipped_at')->nullable();
            $table->timestamp('delivered_at')->nullable();

            // Discount code
            $table->string('coupon_code')->nullable();

            $table->text('notes')->nullable();     // หมายเหตุจากลูกค้า
            $table->text('admin_notes')->nullable(); // หมายเหตุ admin

            $table->timestamps();
            $table->softDeletes();

            $table->index(['user_id', 'status', 'created_at']);
            $table->index(['status', 'paid_at']);
        });

        Schema::create('order_items', function (Blueprint $table) {
            $table->id();
            $table->foreignId('order_id')->constrained()->cascadeOnDelete();
            $table->foreignId('product_id')->constrained()->restrictOnDelete();
            $table->foreignId('variant_id')
                ->nullable()
                ->constrained('product_variants')
                ->nullOnDelete();

            $table->string('product_name');          // Snapshot ชื่อสินค้า
            $table->json('variant_attributes')->nullable(); // Snapshot variant

            $table->unsignedSmallInteger('quantity');
            $table->decimal('unit_price', 10, 2);    // ราคาต่อหน่วย (snapshot)
            $table->decimal('total_price', 10, 2);   // quantity * unit_price

            $table->timestamps();

            $table->index(['order_id', 'product_id']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('order_items');
        Schema::dropIfExists('orders');
    }
};
```

### Step 5: Reviews Migration

```php
// database/migrations/xxxx_create_reviews_table.php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('reviews', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->foreignId('product_id')->constrained()->cascadeOnDelete();
            $table->foreignId('order_item_id')
                ->nullable()
                ->constrained()
                ->nullOnDelete();

            $table->unsignedTinyInteger('rating');  // 1-5
            $table->string('title')->nullable();
            $table->text('comment')->nullable();
            $table->json('images')->nullable();      // Array of image paths

            $table->boolean('is_verified')->default(false); // ซื้อจริง
            $table->boolean('is_approved')->default(true);
            $table->unsignedSmallInteger('helpful_count')->default(0); // คนที่กด helpful

            $table->timestamps();
            $table->softDeletes();

            // แต่ละ user review product ได้ครั้งเดียว (per order item)
            $table->unique(['user_id', 'product_id', 'order_item_id']);
            $table->index(['product_id', 'rating', 'is_approved']);
        });

        // Cart
        Schema::create('cart_items', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->foreignId('product_id')->constrained()->cascadeOnDelete();
            $table->foreignId('variant_id')
                ->nullable()
                ->constrained('product_variants')
                ->nullOnDelete();

            $table->unsignedSmallInteger('quantity')->default(1);
            $table->timestamps();

            $table->unique(['user_id', 'product_id', 'variant_id']);
        });

        // Wishlist
        Schema::create('wishlists', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->foreignId('product_id')->constrained()->cascadeOnDelete();
            $table->timestamps();

            $table->unique(['user_id', 'product_id']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('wishlists');
        Schema::dropIfExists('cart_items');
        Schema::dropIfExists('reviews');
    }
};
```

### Step 6: Factories สำหรับ E-commerce

```php
// database/factories/ProductFactory.php
<?php

namespace Database\Factories;

use App\Models\Category;
use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Str;

class ProductFactory extends Factory
{
    public function definition(): array
    {
        $name  = fake()->words(fake()->numberBetween(2, 5), true);
        $price = fake()->randomFloat(2, 99, 9999);

        return [
            'category_id' => Category::inRandomOrder()->first()?->id,
            'user_id'     => User::factory(),
            'name'        => ucfirst($name),
            'slug'        => Str::slug($name) . '-' . fake()->unique()->numberBetween(1000, 9999),
            'sku'         => strtoupper(fake()->unique()->bothify('##??-####')),
            'description' => fake()->paragraphs(2, true),
            'base_price'  => $price,
            'sale_price'  => fake()->optional(0.3)->randomFloat(2, 50, $price * 0.9),
            'stock'       => fake()->numberBetween(0, 500),
            'status'      => fake()->randomElement(['active', 'active', 'active', 'inactive']),
            'is_featured' => fake()->boolean(20),
            'views'       => fake()->numberBetween(0, 5000),
            'weight'      => fake()->optional()->randomFloat(2, 0.1, 10),
        ];
    }

    public function active(): static
    {
        return $this->state(['status' => 'active']);
    }

    public function featured(): static
    {
        return $this->state(['status' => 'active', 'is_featured' => true]);
    }

    public function outOfStock(): static
    {
        return $this->state(['stock' => 0, 'status' => 'out_of_stock']);
    }
}
```

```php
// database/seeders/EcommerceSeeder.php
<?php

namespace Database\Seeders;

use App\Models\Category;
use App\Models\Product;
use App\Models\User;
use App\Models\Order;
use App\Models\OrderItem;
use App\Models\Review;
use Illuminate\Database\Seeder;

class EcommerceSeeder extends Seeder
{
    public function run(): void
    {
        // สร้าง Categories
        $categories = collect([
            ['name' => 'เสื้อผ้า',    'slug' => 'clothing'],
            ['name' => 'อิเล็กทรอนิกส์', 'slug' => 'electronics'],
            ['name' => 'อาหาร',        'slug' => 'food'],
            ['name' => 'ของเล่น',      'slug' => 'toys'],
            ['name' => 'หนังสือ',      'slug' => 'books'],
        ])->each(fn($cat) => Category::create($cat));

        // สร้าง Admin
        $admin = User::factory()->create([
            'name'  => 'Admin',
            'email' => 'admin@shop.com',
            'role'  => 'admin',
        ]);

        // สร้าง Sellers
        $sellers = User::factory(5)->create(['role' => 'seller']);

        // สร้าง Customers
        $customers = User::factory(50)->create(['role' => 'customer']);

        // สร้าง Products
        $products = Product::factory(100)
            ->active()
            ->recycle($sellers)
            ->create();

        // สร้าง Featured Products
        Product::factory(10)
            ->featured()
            ->recycle($sellers)
            ->create();

        // สร้าง Orders
        $customers->each(function (User $customer) use ($products) {
            $orderCount = fake()->numberBetween(0, 5);

            for ($i = 0; $i < $orderCount; $i++) {
                $items = $products->random(fake()->numberBetween(1, 5));
                $subtotal = 0;

                $order = Order::create([
                    'order_number'      => 'ORD-' . now()->year . '-' . str_pad(fake()->unique()->numberBetween(1, 99999), 6, '0', STR_PAD_LEFT),
                    'user_id'           => $customer->id,
                    'status'            => fake()->randomElement(['pending', 'paid', 'shipped', 'delivered']),
                    'subtotal'          => 0, // จะคำนวณทีหลัง
                    'shipping_fee'      => fake()->randomElement([0, 50, 80, 100]),
                    'total'             => 0,
                    'shipping_name'     => $customer->name,
                    'shipping_phone'    => '08' . fake()->numerify('########'),
                    'shipping_address'  => fake()->address(),
                    'shipping_city'     => fake()->city(),
                    'shipping_province' => 'กรุงเทพมหานคร',
                    'shipping_postal_code' => fake()->postcode(),
                    'payment_method'    => fake()->randomElement(['credit_card', 'promptpay', 'cod']),
                ]);

                foreach ($items as $product) {
                    $qty   = fake()->numberBetween(1, 3);
                    $price = $product->sale_price ?? $product->base_price;

                    OrderItem::create([
                        'order_id'       => $order->id,
                        'product_id'     => $product->id,
                        'product_name'   => $product->name,
                        'quantity'       => $qty,
                        'unit_price'     => $price,
                        'total_price'    => $price * $qty,
                    ]);

                    $subtotal += $price * $qty;
                }

                $order->update([
                    'subtotal' => $subtotal,
                    'total'    => $subtotal + $order->shipping_fee,
                ]);

                // Reviews สำหรับ delivered orders
                if ($order->status === 'delivered') {
                    $order->items->each(function ($item) use ($customer) {
                        Review::create([
                            'user_id'    => $customer->id,
                            'product_id' => $item->product_id,
                            'rating'     => fake()->numberBetween(3, 5),
                            'comment'    => fake()->paragraph(),
                            'is_verified' => true,
                        ]);
                    });
                }
            }
        });

        $this->command->info('✓ E-commerce seeding เสร็จสิ้น!');
        $this->command->table(
            ['Table', 'Count'],
            [
                ['Users',    User::count()],
                ['Products', Product::count()],
                ['Orders',   Order::count()],
                ['Reviews',  Review::count()],
            ]
        );
    }
}
```

### รัน E-commerce Schema

```bash
# รัน migrations ทั้งหมด
php artisan migrate:fresh

# รัน seeders
php artisan db:seed --class=EcommerceSeeder

# หรือรวมเป็นคำสั่งเดียว
php artisan migrate:fresh --seed
```

---

## Quiz

### คำถาม

**1.** Column type ใดที่เหมาะสมที่สุดสำหรับเก็บราคาสินค้า?

a) `$table->float('price')`
b) `$table->double('price')`
c) `$table->decimal('price', 10, 2)`
d) `$table->integer('price')`

**2.** `->cascadeOnDelete()` ทำงานอย่างไร?

a) ป้องกันการลบ parent ถ้ามี child
b) ลบ child records อัตโนมัติเมื่อ parent ถูกลบ
c) Set FK เป็น null เมื่อ parent ถูกลบ
d) ไม่ทำอะไร

**3.** `php artisan migrate:fresh` ต่างจาก `php artisan migrate:refresh` อย่างไร?

a) เหมือนกัน
b) `fresh` ลบทุก table และ migrate ใหม่, `refresh` rollback และ migrate ใหม่
c) `fresh` เร็วกว่า
d) `refresh` ลบ data, `fresh` ไม่ลบ

**4.** Factory `state()` ใช้ทำอะไร?

a) บันทึก state ของ database
b) สร้าง variations ของ factory definition
c) ตรวจสอบ database state
d) Reset factory

**5.** `$table->fullText(['title', 'content'])` ใช้ทำอะไร?

a) สร้าง unique index
b) สร้าง full-text search index สำหรับค้นหา
c) สร้าง composite primary key
d) สร้าง covering index

### เฉลย

| ข้อ | คำตอบ | อธิบาย |
|-----|-------|--------|
| 1 | **c** | `decimal` มีความแม่นยำคงที่ เหมาะสำหรับเงิน, float/double มีปัญหา floating point |
| 2 | **b** | `cascadeOnDelete` ลบ child records ด้วยเมื่อ parent ถูกลบ |
| 3 | **b** | `fresh` drop ทุก table แล้ว migrate ใหม่, `refresh` rollback แต่ละ migration แล้ว re-run |
| 4 | **b** | `state()` สร้าง variation ของ factory ที่ override บาง fields |
| 5 | **b** | `fullText` สร้าง full-text index สำหรับ `whereFullText()` query |

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- การสร้างและโครงสร้าง Migration files
- Column types ทุกประเภทและ Column modifiers
- Foreign Keys และ referential actions (cascade, restrict, null)
- Index types: unique, composite, full-text, spatial
- Migration commands: migrate, rollback, reset, fresh, refresh
- Seeders สำหรับเติมข้อมูลเริ่มต้น
- Factories สำหรับ fake data และ testing
- Workshop: Database schema สมบูรณ์สำหรับ E-commerce

---

## ลิงก์ไป Part ถัดไป

➡️ **[Part 031: Laravel Eloquent ORM](./part-031-laravel-eloquent.md)**

ใน Part ถัดไปเราจะเรียนรู้ Eloquent ORM อย่างละเอียด ตั้งแต่ basic queries, Relationships ทุกประเภท, Eager Loading ไปจนถึง Custom Scopes และ Mutators
