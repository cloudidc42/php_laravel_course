# Part 032: Laravel Eloquent Relationships

## ระดับ: Intermediate
## เวลาที่ใช้: 5-6 ชั่วโมง

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- กำหนด hasOne, belongsTo, hasMany relationships ได้
- ใช้ belongsToMany สำหรับ many-to-many
- ใช้ hasManyThrough สำหรับ nested relationships
- ทำงานกับ Polymorphic relationships
- ใช้ Eager Loading เพื่อแก้ปัญหา N+1
- สร้างระบบ Blog ที่มี Users, Posts, Comments, Tags ครบรูปแบบ

---

## 1. ประเภทของ Relationships

Laravel Eloquent รองรับ relationships หลักๆ ดังนี้:

| Relationship | ความหมาย | ตัวอย่าง |
|-------------|----------|---------|
| `hasOne` | 1 ต่อ 1 | User มี 1 Profile |
| `belongsTo` | หลายต่อ 1 | Profile เป็นของ User |
| `hasMany` | 1 ต่อ หลาย | User มีหลาย Posts |
| `belongsToMany` | หลายต่อหลาย | Post มีหลาย Tags |
| `hasManyThrough` | ผ่าน 3 tables | Country มีหลาย Posts ผ่าน Users |
| `morphTo` | Polymorphic | Comment เป็นของ Post หรือ Video |
| `morphMany` | Polymorphic 1 ต่อ หลาย | Image ของ Post หรือ User |
| `morphToMany` | Polymorphic หลายต่อหลาย | Tag ของ Post หรือ Video |

---

## 2. hasOne / belongsTo

ความสัมพันธ์ 1 ต่อ 1

### 2.1 สร้าง Migrations

```php
<?php
// users table มีอยู่แล้ว

// database/migrations/create_profiles_table.php
Schema::create('profiles', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->unique()->constrained()->onDelete('cascade');
    $table->string('phone')->nullable();
    $table->date('birthday')->nullable();
    $table->string('avatar_url')->nullable();
    $table->text('bio')->nullable();
    $table->string('website')->nullable();
    $table->json('social_links')->nullable();
    $table->timestamps();
});
```

### 2.2 กำหนด Relationships

```php
<?php
// app/Models/User.php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;

class User extends Authenticatable
{
    /**
     * User hasOne Profile
     * ความหมาย: User มี profile 1 อัน
     * FK อยู่ที่ profiles.user_id
     */
    public function profile()
    {
        return $this->hasOne(Profile::class);
        // เทียบเท่า: $this->hasOne(Profile::class, 'user_id', 'id')
        // Parameter: (Model, foreign_key, local_key)
    }
}
```

```php
<?php
// app/Models/Profile.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Profile extends Model
{
    protected $fillable = [
        'user_id', 'phone', 'birthday', 'avatar_url', 'bio', 'website', 'social_links'
    ];

    protected $casts = [
        'birthday'     => 'date',
        'social_links' => 'array',
    ];

    /**
     * Profile belongsTo User
     * ความหมาย: Profile เป็นของ User
     */
    public function user()
    {
        return $this->belongsTo(User::class);
        // เทียบเท่า: $this->belongsTo(User::class, 'user_id', 'id')
        // Parameter: (Model, foreign_key, owner_key)
    }
}
```

### 2.3 การใช้งาน hasOne / belongsTo

```php
<?php
// ดึง profile ของ user
$user = User::find(1);
$profile = $user->profile; // SQL: SELECT * FROM profiles WHERE user_id = 1 LIMIT 1

// ดึง user ของ profile
$profile = Profile::find(1);
$user = $profile->user; // SQL: SELECT * FROM users WHERE id = ? LIMIT 1

// Create associated record
$user = User::find(1);

// วิธีที่ 1: create() - สร้าง profile และ set user_id อัตโนมัติ
$profile = $user->profile()->create([
    'phone' => '0812345678',
    'bio'   => 'Laravel Developer',
]);

// วิธีที่ 2: save() 
$profile = new Profile(['phone' => '0812345678']);
$user->profile()->save($profile);

// วิธีที่ 3: associate() สำหรับ belongsTo
$user = User::find(1);
$profile = Profile::find(5);
$profile->user()->associate($user);
$profile->save();

// Dissociate (set FK เป็น null)
$profile->user()->dissociate();
$profile->save();

// ดึงพร้อม relationship
$user = User::with('profile')->find(1);
echo $user->profile->phone;
echo $user->profile->bio;

// Check if relationship exists
if ($user->profile) {
    echo "มี profile";
} else {
    echo "ยังไม่มี profile";
}
```

---

## 3. hasMany / belongsTo

ความสัมพันธ์ 1 ต่อ หลาย

### 3.1 Migrations

```php
<?php
// database/migrations/create_posts_table.php
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->onDelete('cascade');
    $table->string('title');
    $table->string('slug')->unique();
    $table->text('excerpt')->nullable();
    $table->longText('content');
    $table->string('featured_image')->nullable();
    $table->enum('status', ['draft', 'published', 'archived'])->default('draft');
    $table->timestamp('published_at')->nullable();
    $table->integer('views_count')->default(0);
    $table->timestamps();
});

// database/migrations/create_comments_table.php
Schema::create('comments', function (Blueprint $table) {
    $table->id();
    $table->foreignId('post_id')->constrained()->onDelete('cascade');
    $table->foreignId('user_id')->constrained()->onDelete('cascade');
    $table->unsignedBigInteger('parent_id')->nullable(); // nested comments
    $table->text('content');
    $table->boolean('is_approved')->default(false);
    $table->timestamps();
    $table->foreign('parent_id')->references('id')->on('comments')->onDelete('cascade');
});
```

### 3.2 Models

```php
<?php
// app/Models/User.php

class User extends Authenticatable
{
    /**
     * User hasMany Posts
     */
    public function posts()
    {
        return $this->hasMany(Post::class);
    }

    /**
     * User hasMany Comments
     */
    public function comments()
    {
        return $this->hasMany(Comment::class);
    }

    /**
     * Only published posts
     */
    public function publishedPosts()
    {
        return $this->hasMany(Post::class)->where('status', 'published');
    }
}
```

```php
<?php
// app/Models/Post.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Support\Str;

class Post extends Model
{
    use HasFactory;

    protected $fillable = [
        'user_id', 'title', 'slug', 'excerpt', 'content',
        'featured_image', 'status', 'published_at',
    ];

    protected $casts = [
        'published_at' => 'datetime',
        'views_count'  => 'integer',
    ];

    protected static function booted(): void
    {
        static::creating(function (Post $post) {
            if (empty($post->slug)) {
                $post->slug = Str::slug($post->title);
            }
        });
    }

    /**
     * Post belongsTo User (author)
     */
    public function author()
    {
        return $this->belongsTo(User::class, 'user_id');
        // ใช้ 'user_id' ไม่ใช่ 'author_id' เพราะเราตั้งชื่อ FK เป็น user_id
    }

    /**
     * Post hasMany Comments
     */
    public function comments()
    {
        return $this->hasMany(Comment::class);
    }

    /**
     * เฉพาะ comments ที่ approved
     */
    public function approvedComments()
    {
        return $this->hasMany(Comment::class)->where('is_approved', true);
    }

    // Scopes
    public function scopePublished($query)
    {
        return $query->where('status', 'published')
                     ->whereNotNull('published_at')
                     ->where('published_at', '<=', now());
    }

    public function scopeDraft($query)
    {
        return $query->where('status', 'draft');
    }
}
```

```php
<?php
// app/Models/Comment.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Comment extends Model
{
    protected $fillable = ['post_id', 'user_id', 'parent_id', 'content', 'is_approved'];

    protected $casts = [
        'is_approved' => 'boolean',
    ];

    /**
     * Comment belongsTo Post
     */
    public function post()
    {
        return $this->belongsTo(Post::class);
    }

    /**
     * Comment belongsTo User
     */
    public function author()
    {
        return $this->belongsTo(User::class, 'user_id');
    }

    /**
     * Comment belongsTo parent Comment
     */
    public function parent()
    {
        return $this->belongsTo(Comment::class, 'parent_id');
    }

    /**
     * Comment hasMany child Comments
     */
    public function replies()
    {
        return $this->hasMany(Comment::class, 'parent_id');
    }
}
```

### 3.3 การใช้งาน hasMany

```php
<?php
// ดึงทุก posts ของ user
$user = User::find(1);
$posts = $user->posts; // Collection ของ Post

// ดึง posts ที่ published เท่านั้น
$publishedPosts = $user->publishedPosts;

// นับจำนวน
$postCount = $user->posts()->count();

// ดึงพร้อม order
$latestPosts = $user->posts()->latest()->take(5)->get();

// Create post สำหรับ user
$post = $user->posts()->create([
    'title'   => 'My First Post',
    'content' => 'Hello World!',
    'status'  => 'draft',
]);

// Save post
$post = new Post(['title' => 'New Post', 'content' => '...']);
$user->posts()->save($post);

// ดึง author ของ post
$post = Post::find(1);
$author = $post->author; // User object

// ดึง comments ของ post
$comments = $post->comments;

// ดึง approved comments
$approvedComments = $post->approvedComments;

// สร้าง comment
$comment = $post->comments()->create([
    'user_id' => auth()->id(),
    'content' => 'Great post!',
]);
```

---

## 4. belongsToMany (Many-to-Many)

### 4.1 สร้าง Tags System

```php
<?php
// database/migrations/create_tags_table.php
Schema::create('tags', function (Blueprint $table) {
    $table->id();
    $table->string('name')->unique();
    $table->string('slug')->unique();
    $table->string('color')->nullable();
    $table->timestamps();
});

// database/migrations/create_post_tag_table.php
// Pivot table: ชื่อเป็น singular ทั้งคู่ เรียงตาม alphabetical order
Schema::create('post_tag', function (Blueprint $table) {
    $table->foreignId('post_id')->constrained()->onDelete('cascade');
    $table->foreignId('tag_id')->constrained()->onDelete('cascade');
    $table->primary(['post_id', 'tag_id']); // Composite primary key
    $table->timestamps(); // optional: withTimestamps()
});
```

### 4.2 Models

```php
<?php
// app/Models/Post.php

class Post extends Model
{
    /**
     * Post belongsToMany Tags
     */
    public function tags()
    {
        return $this->belongsToMany(Tag::class);
        // เทียบเท่า:
        // $this->belongsToMany(Tag::class, 'post_tag', 'post_id', 'tag_id')
        // Parameter: (Model, pivot_table, foreign_key, related_key)
    }

    /**
     * belongsToMany พร้อม extra columns ใน pivot table
     */
    public function tagsWithOrder()
    {
        return $this->belongsToMany(Tag::class)
            ->withPivot('sort_order')         // ดึง extra columns จาก pivot
            ->withTimestamps()                  // ดึง created_at, updated_at จาก pivot
            ->orderBy('sort_order');
    }
}
```

```php
<?php
// app/Models/Tag.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Support\Str;

class Tag extends Model
{
    use HasFactory;

    protected $fillable = ['name', 'slug', 'color'];

    protected static function booted(): void
    {
        static::creating(function (Tag $tag) {
            if (empty($tag->slug)) {
                $tag->slug = Str::slug($tag->name);
            }
        });
    }

    /**
     * Tag belongsToMany Posts
     */
    public function posts()
    {
        return $this->belongsToMany(Post::class);
    }
}
```

### 4.3 การใช้งาน belongsToMany

```php
<?php
$post = Post::find(1);

// ดึง tags ของ post
$tags = $post->tags; // Collection ของ Tag

// Attach - เพิ่ม tags
$post->tags()->attach(1);                     // attach tag id 1
$post->tags()->attach([1, 2, 3]);             // attach หลาย tags
$post->tags()->attach([1 => ['sort_order' => 1]]); // attach พร้อม pivot data

// Detach - ลบ tags
$post->tags()->detach(1);          // ลบ tag id 1
$post->tags()->detach([1, 2, 3]);  // ลบหลาย tags
$post->tags()->detach();           // ลบทุก tags

// Sync - เซ็ต tags ใหม่ทั้งหมด (ลบเก่า, เพิ่มใหม่)
$post->tags()->sync([1, 2, 3]);

// Sync โดยไม่ลบ tags เดิม
$post->tags()->syncWithoutDetaching([4, 5]);

// Toggle - ถ้ามีแล้ว ลบ, ถ้าไม่มี เพิ่ม
$post->tags()->toggle([1, 2, 3]);
$post->tags()->toggle(1);

// เช็คว่ามี tag หรือไม่
$hasPhpTag = $post->tags()->where('name', 'PHP')->exists();

// ดึงพร้อม pivot data
$tags = $post->tags()->withPivot('sort_order', 'created_at')->get();
foreach ($tags as $tag) {
    echo $tag->name;
    echo $tag->pivot->sort_order; // ข้อมูลจาก pivot table
}

// ค้นหา posts ที่มี tag ใด tag หนึ่ง
$phpPosts = Post::whereHas('tags', function ($query) {
    $query->where('name', 'PHP');
})->get();

// ค้นหา posts ที่มี tags ทั้งหมดที่ระบุ
$posts = Post::whereHas('tags', function ($query) {
    $query->where('name', 'PHP');
})->whereHas('tags', function ($query) {
    $query->where('name', 'Laravel');
})->get();
```

### 4.4 Custom Pivot Model

```php
<?php
// app/Models/PostTag.php (Pivot Model)

namespace App\Models;

use Illuminate\Database\Eloquent\Relations\Pivot;

class PostTag extends Pivot
{
    protected $table = 'post_tag';
    
    public $timestamps = true;

    protected $fillable = ['post_id', 'tag_id', 'sort_order'];
    
    protected $casts = [
        'sort_order' => 'integer',
    ];

    // สามารถเพิ่ม methods ใน pivot ได้
    public function post()
    {
        return $this->belongsTo(Post::class);
    }

    public function tag()
    {
        return $this->belongsTo(Tag::class);
    }
}
```

```php
<?php
// app/Models/Post.php

class Post extends Model
{
    public function tags()
    {
        return $this->belongsToMany(Tag::class)
            ->using(PostTag::class)  // ใช้ custom pivot model
            ->withTimestamps()
            ->withPivot('sort_order');
    }
}
```

---

## 5. hasManyThrough

ดึงข้อมูลผ่านความสัมพันธ์ 3 ชั้น

```
Country -> User -> Post
```

### 5.1 Migrations

```php
<?php
// database/migrations/create_countries_table.php
Schema::create('countries', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('code', 2)->unique();
    $table->timestamps();
});

// เพิ่ม country_id ใน users table
Schema::table('users', function (Blueprint $table) {
    $table->foreignId('country_id')->nullable()->constrained()->onDelete('set null');
});
```

### 5.2 Model

```php
<?php
// app/Models/Country.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Country extends Model
{
    protected $fillable = ['name', 'code'];

    /**
     * Country has many Users
     */
    public function users()
    {
        return $this->hasMany(User::class);
    }

    /**
     * Country hasManyThrough Posts (ผ่าน Users)
     * ดึง posts ทั้งหมดของ users ในประเทศนี้
     */
    public function posts()
    {
        return $this->hasManyThrough(
            Post::class,    // Final model ที่ต้องการ
            User::class,    // Intermediate model
            'country_id',   // FK บน intermediate (users.country_id)
            'user_id',      // FK บน final (posts.user_id)
            'id',           // Local key บน source (countries.id)
            'id'            // Local key บน intermediate (users.id)
        );
    }

    /**
     * hasOneThrough: Country hasOneThrough (latest post)
     */
    public function latestPost()
    {
        return $this->hasOneThrough(Post::class, User::class)
            ->latestOfMany();
    }
}
```

### 5.3 การใช้งาน

```php
<?php
$thailand = Country::where('code', 'TH')->first();

// ดึงทุก posts ของ users ในประเทศไทย
$thaiPosts = $thailand->posts;

// นับ
$postCount = $thailand->posts()->count();

// Filter
$publishedThaiPosts = $thailand->posts()->where('status', 'published')->get();
```

---

## 6. Polymorphic Relationships

ใช้เมื่อ model เดียวสามารถเป็นของหลาย model ต่างชนิด

### 6.1 morphTo / morphMany (1 ต่อ หลาย)

ตัวอย่าง: Comment สามารถเป็นของ Post หรือ Video ก็ได้

```php
<?php
// database/migrations/modify_comments_table.php
// เปลี่ยน post_id เป็น commentable_id + commentable_type
Schema::table('comments', function (Blueprint $table) {
    $table->dropForeign(['post_id']);
    $table->dropColumn('post_id');
    $table->unsignedBigInteger('commentable_id');
    $table->string('commentable_type');
    // Index สำหรับ polymorphic
    $table->index(['commentable_id', 'commentable_type']);
});
```

```php
<?php
// app/Models/Comment.php

class Comment extends Model
{
    /**
     * Comment morphTo (เป็นของ Polymorphic)
     * จะดูจาก commentable_type ว่าเป็น Model ไหน
     */
    public function commentable()
    {
        return $this->morphTo();
        // เทียบเท่า: $this->morphTo(__FUNCTION__, 'commentable_type', 'commentable_id')
    }
}
```

```php
<?php
// app/Models/Post.php

class Post extends Model
{
    /**
     * Post morphMany Comments
     */
    public function comments()
    {
        return $this->morphMany(Comment::class, 'commentable');
        // Parameter: (Model, morphName)
        // morphName ต้องตรงกับชื่อใน Comment::commentable()
    }
}
```

```php
<?php
// app/Models/Video.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Video extends Model
{
    protected $fillable = ['title', 'url', 'duration'];

    /**
     * Video morphMany Comments
     */
    public function comments()
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}
```

### 6.2 การใช้งาน Polymorphic

```php
<?php
// ดึง comments ของ post
$post = Post::find(1);
$comments = $post->comments;

// ดึง comments ของ video
$video = Video::find(1);
$comments = $video->comments;

// สร้าง comment สำหรับ post
$post->comments()->create([
    'user_id' => auth()->id(),
    'content' => 'Great article!',
]);

// สร้าง comment สำหรับ video
$video->comments()->create([
    'user_id' => auth()->id(),
    'content' => 'Awesome video!',
]);

// ดึง parent ของ comment
$comment = Comment::find(1);
$parent = $comment->commentable; // จะ return Post หรือ Video

// เช็คว่า parent เป็น model ไหน
if ($comment->commentable instanceof Post) {
    echo "Comment บน Post: " . $comment->commentable->title;
} elseif ($comment->commentable instanceof Video) {
    echo "Comment บน Video: " . $comment->commentable->title;
}

// ข้อมูลใน database:
// comments table:
// | id | commentable_id | commentable_type | content |
// |  1 |              1 | App\Models\Post  | Great!  |
// |  2 |              1 | App\Models\Video | Awesome!|
```

### 6.3 morphToMany (Many-to-Many Polymorphic)

ตัวอย่าง: Tag สามารถ tag Post หรือ Video ก็ได้

```php
<?php
// database/migrations/create_taggables_table.php
Schema::create('taggables', function (Blueprint $table) {
    $table->foreignId('tag_id')->constrained()->onDelete('cascade');
    $table->unsignedBigInteger('taggable_id');
    $table->string('taggable_type');
    $table->index(['taggable_id', 'taggable_type']);
});
```

```php
<?php
// app/Models/Tag.php

class Tag extends Model
{
    /**
     * Tag morphedByMany Posts
     */
    public function posts()
    {
        return $this->morphedByMany(Post::class, 'taggable');
    }

    /**
     * Tag morphedByMany Videos
     */
    public function videos()
    {
        return $this->morphedByMany(Video::class, 'taggable');
    }
}
```

```php
<?php
// app/Models/Post.php

class Post extends Model
{
    /**
     * Post morphToMany Tags
     */
    public function tags()
    {
        return $this->morphToMany(Tag::class, 'taggable');
    }
}
```

```php
<?php
// app/Models/Video.php

class Video extends Model
{
    public function tags()
    {
        return $this->morphToMany(Tag::class, 'taggable');
    }
}
```

```php
<?php
// การใช้งาน
$post = Post::find(1);
$post->tags()->attach([1, 2, 3]);
$post->tags()->sync([2, 3, 4]);
$tags = $post->tags;

$video = Video::find(1);
$video->tags()->attach([1, 5]);
$tags = $video->tags;

$tag = Tag::find(1);
$taggedPosts = $tag->posts;
$taggedVideos = $tag->videos;
```

---

## 7. Eager Loading

### 7.1 ปัญหา N+1

```php
<?php
// BAD: N+1 Problem
// ถ้ามี 100 posts จะทำ 101 queries (1 + 100)
$posts = Post::all();
foreach ($posts as $post) {
    echo $post->author->name; // Query ใหม่ต่อ post!
}
```

SQL ที่เกิดขึ้น:
```sql
SELECT * FROM posts;                    -- Query #1
SELECT * FROM users WHERE id = 1;       -- Query #2
SELECT * FROM users WHERE id = 2;       -- Query #3
-- ... ทำซ้ำ N ครั้ง
```

### 7.2 Eager Loading ด้วย with()

```php
<?php
// GOOD: Eager Loading - ทำแค่ 2 queries เสมอ
$posts = Post::with('author')->get();
foreach ($posts as $post) {
    echo $post->author->name; // ไม่มี query ใหม่!
}
```

SQL ที่เกิดขึ้น:
```sql
SELECT * FROM posts;
SELECT * FROM users WHERE id IN (1, 2, 3, ...);  -- Query เดียวสำหรับทุก users
```

### 7.3 Eager Loading หลาย Relationships

```php
<?php
// Load หลาย relationships พร้อมกัน
$posts = Post::with(['author', 'tags', 'comments'])->get();

// Nested eager loading (load relationship ของ relationship)
$posts = Post::with([
    'author',
    'author.profile',     // load profile ของ author ด้วย
    'comments',
    'comments.author',    // load author ของแต่ละ comment
    'tags',
])->get();

// Eager loading พร้อม conditions
$posts = Post::with([
    'comments' => function ($query) {
        $query->where('is_approved', true)
              ->orderBy('created_at', 'desc')
              ->limit(5);
    },
    'tags' => function ($query) {
        $query->select('tags.id', 'tags.name'); // เลือกเฉพาะ columns ที่ต้องการ
    },
])->get();

// Eager loading ด้วย withCount
$posts = Post::withCount(['comments', 'tags'])->get();
foreach ($posts as $post) {
    echo $post->comments_count; // ไม่ต้อง query ใหม่
    echo $post->tags_count;
}

// withSum, withAvg, withMax, withMin
$posts = Post::withSum('comments', 'likes_count')
    ->withAvg('reviews', 'rating')
    ->get();

// withExists
$posts = Post::withExists('comments')->get();
foreach ($posts as $post) {
    echo $post->comments_exists ? 'มี comments' : 'ไม่มี comments';
}
```

### 7.4 Lazy Eager Loading ด้วย load()

```php
<?php
// load() ใช้หลังจาก query แล้ว
$posts = Post::all();

// โหลด relationship ทีหลัง
$posts->load('author');
$posts->load(['author', 'tags', 'comments']);

// loadMissing - โหลดเฉพาะที่ยังไม่ได้โหลด
$posts->loadMissing('author');

// loadCount
$posts->loadCount('comments');

// สำหรับ single model
$post = Post::find(1);
$post->load(['author', 'tags', 'comments']);
```

### 7.5 Preventing Lazy Loading

```php
<?php
// app/Providers/AppServiceProvider.php

use Illuminate\Database\Eloquent\Model;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Development: throw exception เมื่อมี lazy loading
        // จะช่วย detect N+1 ได้
        Model::preventLazyLoading(!app()->isProduction());
    }
}
```

---

## 8. Workshop: Blog System

### 8.1 โครงสร้างระบบ

```
Blog System
├── users (id, name, email, country_id)
├── profiles (id, user_id, bio, avatar_url)
├── countries (id, name, code)
├── categories (id, name, slug, parent_id)
├── posts (id, user_id, category_id, title, content, status)
├── comments (id, user_id, commentable_id, commentable_type, content)
├── tags (id, name, slug)
└── taggables (tag_id, taggable_id, taggable_type)
```

### 8.2 Complete Blog Models

```php
<?php
// app/Models/User.php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use Notifiable;

    protected $fillable = ['name', 'email', 'password', 'country_id'];

    protected $hidden = ['password', 'remember_token'];

    protected $casts = [
        'email_verified_at' => 'datetime',
        'password'          => 'hashed',
    ];

    // Relationships
    public function profile()
    {
        return $this->hasOne(Profile::class);
    }

    public function country()
    {
        return $this->belongsTo(Country::class);
    }

    public function posts()
    {
        return $this->hasMany(Post::class);
    }

    public function comments()
    {
        return $this->hasMany(Comment::class);
    }

    // Helper methods
    public function getAvatarUrlAttribute(): string
    {
        return $this->profile?->avatar_url
            ?? 'https://ui-avatars.com/api/?name=' . urlencode($this->name);
    }
}
```

```php
<?php
// app/Models/Post.php (ฉบับสมบูรณ์)

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Support\Str;

class Post extends Model
{
    use HasFactory;

    protected $fillable = [
        'user_id', 'category_id', 'title', 'slug', 'excerpt',
        'content', 'featured_image', 'status', 'published_at',
    ];

    protected $casts = [
        'published_at' => 'datetime',
        'views_count'  => 'integer',
    ];

    protected static function booted(): void
    {
        static::creating(function (Post $post) {
            if (empty($post->slug)) {
                $post->slug = Str::slug($post->title);
            }
        });
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

    public function comments()
    {
        return $this->morphMany(Comment::class, 'commentable')
            ->orderBy('created_at', 'desc');
    }

    public function approvedComments()
    {
        return $this->morphMany(Comment::class, 'commentable')
            ->where('is_approved', true)
            ->orderBy('created_at', 'desc');
    }

    public function tags()
    {
        return $this->morphToMany(Tag::class, 'taggable');
    }

    // Scopes
    public function scopePublished(Builder $query): Builder
    {
        return $query->where('status', 'published')
                     ->whereNotNull('published_at')
                     ->where('published_at', '<=', now());
    }

    public function scopeByAuthor(Builder $query, int $userId): Builder
    {
        return $query->where('user_id', $userId);
    }

    public function scopeByCategory(Builder $query, int $categoryId): Builder
    {
        return $query->where('category_id', $categoryId);
    }

    public function scopeSearch(Builder $query, string $term): Builder
    {
        return $query->where(function ($q) use ($term) {
            $q->where('title', 'like', "%{$term}%")
              ->orWhere('content', 'like', "%{$term}%")
              ->orWhere('excerpt', 'like', "%{$term}%");
        });
    }

    // Methods
    public function publish(): bool
    {
        return $this->update([
            'status'       => 'published',
            'published_at' => $this->published_at ?? now(),
        ]);
    }

    public function incrementViews(): void
    {
        $this->increment('views_count');
    }

    public function isPublished(): bool
    {
        return $this->status === 'published' 
            && $this->published_at !== null 
            && $this->published_at->isPast();
    }
}
```

### 8.3 Blog Controller

```php
<?php
// app/Http/Controllers/Blog/PostController.php

namespace App\Http\Controllers\Blog;

use App\Http\Controllers\Controller;
use App\Models\Post;
use App\Models\Category;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class PostController extends Controller
{
    /**
     * ดึงรายการ posts พร้อม relationships
     */
    public function index(Request $request): JsonResponse
    {
        $query = Post::published()
            ->with([
                'author:id,name',          // eager load เฉพาะ id, name
                'author.profile:id,user_id,avatar_url',
                'category:id,name,slug',
                'tags:id,name,slug,color',
            ])
            ->withCount(['comments', 'tags']);

        if ($request->has('category')) {
            $query->byCategory($request->integer('category'));
        }

        if ($request->has('author')) {
            $query->byAuthor($request->integer('author'));
        }

        if ($request->has('tag')) {
            $query->whereHas('tags', function ($q) use ($request) {
                $q->where('slug', $request->string('tag'));
            });
        }

        if ($request->has('search')) {
            $query->search($request->string('search'));
        }

        $posts = $query->latest('published_at')->paginate(10);

        return response()->json($posts);
    }

    /**
     * แสดง post เดี่ยวพร้อม comments
     */
    public function show(string $slug): JsonResponse
    {
        $post = Post::published()
            ->where('slug', $slug)
            ->with([
                'author:id,name',
                'author.profile:id,user_id,avatar_url,bio',
                'category:id,name,slug',
                'tags:id,name,slug,color',
                'approvedComments' => function ($query) {
                    $query->with([
                        'author:id,name',
                        'replies.author:id,name',
                    ])
                    ->whereNull('parent_id')
                    ->limit(20);
                },
            ])
            ->withCount(['comments', 'tags'])
            ->firstOrFail();

        $post->incrementViews();

        return response()->json($post);
    }

    /**
     * สร้าง post ใหม่
     */
    public function store(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'title'        => 'required|string|max:255',
            'content'      => 'required|string',
            'excerpt'      => 'nullable|string|max:500',
            'category_id'  => 'required|exists:categories,id',
            'tags'         => 'array',
            'tags.*'       => 'exists:tags,id',
            'status'       => 'in:draft,published',
            'published_at' => 'nullable|date',
        ]);

        $post = $request->user()->posts()->create($validated);

        // Attach tags
        if (!empty($validated['tags'])) {
            $post->tags()->sync($validated['tags']);
        }

        $post->load(['author', 'category', 'tags']);

        return response()->json([
            'message' => 'สร้าง post สำเร็จ',
            'post'    => $post,
        ], 201);
    }
}
```

### 8.4 Comment Controller

```php
<?php
// app/Http/Controllers/Blog/CommentController.php

namespace App\Http\Controllers\Blog;

use App\Http\Controllers\Controller;
use App\Models\Comment;
use App\Models\Post;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class CommentController extends Controller
{
    /**
     * สร้าง comment สำหรับ post
     */
    public function store(Request $request, int $postId): JsonResponse
    {
        $post = Post::published()->findOrFail($postId);

        $validated = $request->validate([
            'content'   => 'required|string|max:1000',
            'parent_id' => 'nullable|exists:comments,id',
        ]);

        // ตรวจสอบว่า parent comment เป็นของ post นี้
        if (!empty($validated['parent_id'])) {
            $parentComment = Comment::where('id', $validated['parent_id'])
                ->where('commentable_id', $postId)
                ->where('commentable_type', Post::class)
                ->firstOrFail();
        }

        $comment = $post->comments()->create([
            'user_id'   => $request->user()->id,
            'content'   => $validated['content'],
            'parent_id' => $validated['parent_id'] ?? null,
        ]);

        $comment->load('author:id,name');

        return response()->json([
            'message' => 'เพิ่ม comment สำเร็จ',
            'comment' => $comment,
        ], 201);
    }
}
```

### 8.5 Tag Controller

```php
<?php
// app/Http/Controllers/Blog/TagController.php

namespace App\Http\Controllers\Blog;

use App\Http\Controllers\Controller;
use App\Models\Tag;
use Illuminate\Http\JsonResponse;

class TagController extends Controller
{
    /**
     * ดึง posts ของ tag
     */
    public function posts(string $slug): JsonResponse
    {
        $tag = Tag::where('slug', $slug)->firstOrFail();

        $posts = $tag->posts()
            ->published()
            ->with(['author:id,name', 'category:id,name,slug'])
            ->withCount('comments')
            ->latest('published_at')
            ->paginate(15);

        return response()->json([
            'tag'   => $tag,
            'posts' => $posts,
        ]);
    }

    /**
     * Popular tags
     */
    public function popular(): JsonResponse
    {
        $tags = Tag::withCount([
            'posts' => function ($query) {
                $query->published();
            }
        ])
        ->orderByDesc('posts_count')
        ->limit(20)
        ->get();

        return response()->json($tags);
    }
}
```

---

## 9. Query Optimization

### 9.1 Select เฉพาะ columns ที่ต้องการ

```php
<?php
// BAD: ดึงทุก columns
$posts = Post::with('author')->get();

// GOOD: เลือกเฉพาะที่ต้องการ
$posts = Post::select('id', 'title', 'slug', 'published_at', 'user_id')
    ->with('author:id,name,email')
    ->get();
```

### 9.2 ใช้ whereHas vs Join

```php
<?php
// whereHas - ง่ายกว่า แต่ subquery
$posts = Post::whereHas('tags', function ($query) {
    $query->where('name', 'PHP');
})->get();

// join - เร็วกว่าสำหรับ large datasets
$posts = Post::join('taggables', function ($join) {
    $join->on('posts.id', '=', 'taggables.taggable_id')
         ->where('taggables.taggable_type', Post::class);
})->join('tags', 'taggables.tag_id', '=', 'tags.id')
  ->where('tags.name', 'PHP')
  ->select('posts.*')
  ->distinct()
  ->get();
```

### 9.3 Database Indexes

```php
<?php
// เพิ่ม indexes ที่ใช้บ่อย
Schema::table('posts', function (Blueprint $table) {
    $table->index('status');
    $table->index('user_id');
    $table->index('category_id');
    $table->index('published_at');
    $table->index(['status', 'published_at']); // Composite index
});

Schema::table('comments', function (Blueprint $table) {
    $table->index(['commentable_id', 'commentable_type']);
    $table->index('parent_id');
    $table->index('is_approved');
});
```

---

## Quiz พร้อมเฉลย

### คำถามที่ 1
จงอธิบายความแตกต่างระหว่าง `with()` และ `load()`

**เฉลย:**
- `with()` ใช้ **ก่อน** query (Eager Loading) - จะรวม query เป็น 1+N ที่กำหนดไว้ล่วงหน้า
- `load()` ใช้ **หลัง** query (Lazy Eager Loading) - โหลด relationship หลังจาก query แล้ว
- ใช้ `with()` เมื่อรู้ล่วงหน้าว่าต้องการ relationship ใด
- ใช้ `load()` เมื่อต้องการ relationship แบบ conditional หลัง query

### คำถามที่ 2
อธิบาย N+1 Problem และวิธีแก้

**เฉลย:**
N+1 Problem คือเมื่อดึงข้อมูล N records แล้วทำ query เพิ่มอีก N ครั้งสำหรับ relationship

```php
// BAD: N+1 Problem (101 queries สำหรับ 100 posts)
$posts = Post::all();
foreach ($posts as $post) {
    echo $post->author->name; // query ใหม่ทุกครั้ง
}

// GOOD: Eager Loading (2 queries เสมอ)
$posts = Post::with('author')->get();
foreach ($posts as $post) {
    echo $post->author->name; // ไม่มี query ใหม่
}
```

### คำถามที่ 3
เมื่อไหรควรใช้ Polymorphic Relationships?

**เฉลย:**
ใช้เมื่อ:
1. Model หนึ่งต้องการ belong to หลาย model ต่างชนิด (เช่น Comment ของทั้ง Post และ Video)
2. หลีกเลี่ยงการสร้างหลาย foreign key columns
3. ต้องการ reusable feature ข้าม models

### คำถามที่ 4
เขียน query เพื่อดึง Users ที่มีมากกว่า 5 posts ที่ published

**เฉลย:**
```php
// วิธีที่ 1: withCount + having
$users = User::withCount(['posts' => function ($query) {
    $query->published();
}])
->having('posts_count', '>', 5)
->get();

// วิธีที่ 2: whereHas
$users = User::whereHas('posts', function ($query) {
    $query->published();
}, '>', 5)->get();
```

### คำถามที่ 5
อธิบาย belongsToMany และ hasManyThrough ต่างกันอย่างไร?

**เฉลย:**
- `belongsToMany`: Many-to-Many ผ่าน pivot table (Posts <-> Tags ผ่าน post_tag)
- `hasManyThrough`: ดึงข้อมูลผ่านชั้น intermediate (Country -> Posts ผ่าน Users)
- `belongsToMany` ต้องการ pivot table ที่มี FK ทั้งสองฝั่ง
- `hasManyThrough` ไม่มี pivot table แต่ผ่าน FK ตามลำดับ

---

## แบบฝึกหัด

### Exercise 1: Bookmarks
ผู้ใช้สามารถ bookmark posts ได้ สร้าง:
- Migration สำหรับ bookmarks table
- Relationship บน User และ Post
- Method สำหรับ toggle bookmark

### Exercise 2: Post Likes
สร้างระบบ like สำหรับ posts:
- User สามารถ like post ได้เพียงครั้งเดียว
- นับจำนวน likes
- เช็คว่า user ได้ like แล้วหรือไม่

### Exercise 3: Related Posts
ดึง related posts โดยใช้ tags ร่วมกัน:
```php
// เขียน method ใน Post model
public function relatedPosts(): Collection
{
    // ดึง posts ที่มี tags ร่วมกัน เรียงตามจำนวน tags ที่ match
}
```

---

## สรุป

| Relationship | Method | ใช้เมื่อ |
|-------------|--------|---------|
| 1-to-1 | `hasOne()` | User มี 1 Profile |
| 1-to-1 (reverse) | `belongsTo()` | Profile เป็นของ User |
| 1-to-many | `hasMany()` | User มีหลาย Posts |
| Many-to-many | `belongsToMany()` | Post มีหลาย Tags |
| Through | `hasManyThrough()` | Country มีหลาย Posts ผ่าน Users |
| Polymorphic | `morphMany()/morphTo()` | Comment ของ Post/Video |
| Eager Loading | `with()` | ป้องกัน N+1 |
| Lazy Eager | `load()` | โหลด relationship ทีหลัง |

---

## ลิงก์ไป Part ถัดไป

➡️ [Part 033: Laravel Validation](part-033-laravel-validation.md)

ใน Part ถัดไปเราจะเรียนเรื่อง Validation ซึ่งเป็นสิ่งสำคัญมากในการรับข้อมูลจาก user
