# Part 026: Laravel Installation และการตั้งค่า

**ระดับ: กลาง (Intermediate)**

---

## เป้าหมายของ Part นี้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- ติดตั้ง Laravel ผ่าน Composer ได้อย่างถูกต้อง
- เข้าใจโครงสร้างโปรเจกต์ Laravel ทุกไดเรกทอรี
- ตั้งค่า `.env` และ environment variables
- ใช้งาน Artisan CLI เบื้องต้นได้
- สร้าง Laravel project แรกของตัวเองได้

---

## 1. ข้อกำหนดระบบ (System Requirements)

ก่อนติดตั้ง Laravel คุณต้องมีสิ่งต่อไปนี้:

### PHP Requirements
Laravel ต้องการ **PHP 8.1 ขึ้นไป** (Laravel 10.x)

```bash
# ตรวจสอบ PHP version
php --version
# ควรได้ผลลัพธ์ประมาณ: PHP 8.1.x หรือสูงกว่า
```

### PHP Extensions ที่จำเป็น

| Extension | คำอธิบาย |
|-----------|----------|
| BCMath | การคำนวณตัวเลขแม่นยำสูง |
| Ctype | ตรวจสอบ character type |
| cURL | HTTP requests |
| DOM | XML manipulation |
| Fileinfo | ข้อมูลไฟล์ |
| JSON | encode/decode JSON |
| Mbstring | Multi-byte strings ภาษาไทย |
| OpenSSL | การเข้ารหัส |
| PCRE | Regular expressions |
| PDO | Database abstraction |
| Tokenizer | PHP source tokenizer |
| XML | XML processing |

```bash
# ตรวจสอบ extensions ที่ติดตั้ง
php -m

# ตรวจสอบ extension เฉพาะอัน
php -m | grep -i mbstring
php -m | grep -i openssl
php -m | grep -i pdo
```

### ติดตั้ง PHP Extensions (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install php8.1-cli php8.1-common php8.1-mysql php8.1-zip \
    php8.1-gd php8.1-mbstring php8.1-curl php8.1-xml php8.1-bcmath \
    php8.1-intl php8.1-readline
```

### ติดตั้ง PHP Extensions (macOS with Homebrew)

```bash
brew install php
# Extensions ส่วนใหญ่จะมาพร้อมกัน
```

### ติดตั้ง PHP Extensions (Windows)

```bash
# ใช้ Laragon หรือ XAMPP ที่มาพร้อม extensions ครบแล้ว
# หรือใช้ Windows Subsystem for Linux (WSL2)
```

---

## 2. ติดตั้ง Composer

Composer คือ package manager ของ PHP ที่ Laravel ต้องการ

### Linux/macOS

```bash
# ดาวน์โหลดและติดตั้ง Composer
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php composer-setup.php
php -r "unlink('composer-setup.php');"
sudo mv composer.phar /usr/local/bin/composer

# ตรวจสอบการติดตั้ง
composer --version
# Composer version 2.x.x
```

### Windows

ดาวน์โหลด Composer-Setup.exe จาก https://getcomposer.org/download/

```bash
# ตรวจสอบหลังติดตั้ง
composer --version
```

---

## 3. ติดตั้ง Laravel ผ่าน Composer

### วิธีที่ 1: ใช้ Laravel Installer (แนะนำ)

```bash
# ติดตั้ง Laravel Installer globally
composer global require laravel/installer

# ตรวจสอบว่า PATH มี Composer global bin directory
# เพิ่มใน ~/.bashrc หรือ ~/.zshrc:
export PATH="$HOME/.composer/vendor/bin:$PATH"
# หรือ
export PATH="$HOME/.config/composer/vendor/bin:$PATH"

# สร้างโปรเจกต์ใหม่
laravel new my-blog

# เข้าไปในโปรเจกต์
cd my-blog

# รัน development server
php artisan serve
```

### วิธีที่ 2: ใช้ Composer Create-Project

```bash
# สร้างโปรเจกต์ Laravel
composer create-project laravel/laravel my-blog

# หรือระบุ version
composer create-project laravel/laravel:^10.0 my-blog

# เข้าไปในโปรเจกต์
cd my-blog

# รัน development server
php artisan serve
# Server running at http://127.0.0.1:8000
```

### เปิด Browser ตรวจสอบ

เปิด http://localhost:8000 ควรเห็นหน้า Laravel welcome page

---

## 4. โครงสร้างโปรเจกต์ Laravel

หลังสร้างโปรเจกต์แล้ว จะเห็นโครงสร้างแบบนี้:

```
my-blog/
├── app/                    # โค้ดหลักของแอปพลิเคชัน
│   ├── Console/            # Artisan commands
│   ├── Exceptions/         # Exception handlers
│   ├── Http/               # Controllers, Middleware, Requests
│   │   ├── Controllers/    # Controllers
│   │   ├── Middleware/     # HTTP Middleware
│   │   └── Requests/       # Form Requests
│   ├── Models/             # Eloquent Models
│   └── Providers/          # Service Providers
│
├── bootstrap/              # Application bootstrapping
│   ├── app.php             # Application instance
│   └── cache/              # Framework cache
│
├── config/                 # Configuration files
│   ├── app.php             # Application config
│   ├── auth.php            # Authentication config
│   ├── cache.php           # Cache config
│   ├── database.php        # Database config
│   ├── filesystems.php     # File storage config
│   ├── mail.php            # Mail config
│   ├── queue.php           # Queue config
│   └── session.php         # Session config
│
├── database/               # Database migrations, seeders
│   ├── factories/          # Model Factories
│   ├── migrations/         # Migration files
│   └── seeders/            # Database Seeders
│
├── lang/                   # Language files
│   └── en/                 # English translations
│
├── public/                 # Web root (document root)
│   ├── index.php           # Application entry point
│   ├── .htaccess           # Apache rewrite rules
│   └── favicon.ico         # Favicon
│
├── resources/              # Frontend resources
│   ├── css/                # CSS files
│   ├── js/                 # JavaScript files
│   └── views/              # Blade templates
│
├── routes/                 # Route definitions
│   ├── api.php             # API routes
│   ├── channels.php        # Broadcast channels
│   ├── console.php         # Console routes
│   └── web.php             # Web routes
│
├── storage/                # File storage
│   ├── app/                # Application files
│   ├── framework/          # Framework files (cache, sessions, views)
│   └── logs/               # Log files
│
├── tests/                  # Automated tests
│   ├── Feature/            # Feature tests
│   └── Unit/               # Unit tests
│
├── vendor/                 # Composer packages (อย่า edit!)
│
├── .env                    # Environment variables (อย่า commit!)
├── .env.example            # Example environment file
├── .gitignore              # Git ignore rules
├── artisan                 # Artisan CLI
├── composer.json           # Composer dependencies
├── composer.lock           # Locked dependency versions
├── package.json            # NPM dependencies
└── vite.config.js          # Vite configuration
```

### อธิบายไดเรกทอรีสำคัญ

#### `app/` - โค้ดหลัก

```php
// app/Models/User.php - Model ตัวอย่าง
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use HasFactory, Notifiable;

    protected $fillable = [
        'name',
        'email',
        'password',
    ];

    protected $hidden = [
        'password',
        'remember_token',
    ];

    protected $casts = [
        'email_verified_at' => 'datetime',
        'password' => 'hashed',
    ];
}
```

#### `routes/web.php` - Web Routes

```php
<?php

use Illuminate\Support\Facades\Route;

// Route พื้นฐาน
Route::get('/', function () {
    return view('welcome');
});
```

#### `config/` - Configuration

```php
// config/app.php - ตัวอย่าง config บางส่วน
return [
    'name' => env('APP_NAME', 'Laravel'),
    'env' => env('APP_ENV', 'production'),
    'debug' => (bool) env('APP_DEBUG', false),
    'url' => env('APP_URL', 'http://localhost'),
    'timezone' => 'UTC',
    'locale' => 'en',
    // ...
];
```

---

## 5. .env Configuration

ไฟล์ `.env` เก็บ environment variables ที่แตกต่างกันในแต่ละ environment

### โครงสร้าง .env

```bash
# ไฟล์ .env ตัวอย่างสมบูรณ์
APP_NAME="My Blog"
APP_ENV=local
APP_KEY=base64:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
APP_DEBUG=true
APP_URL=http://localhost:8000

LOG_CHANNEL=stack
LOG_DEPRECATIONS_CHANNEL=null
LOG_LEVEL=debug

# Database Configuration
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=my_blog
DB_USERNAME=root
DB_PASSWORD=secret

# Redis (Cache/Session/Queue)
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

# Cache
CACHE_DRIVER=file
QUEUE_CONNECTION=sync
SESSION_DRIVER=file
SESSION_LIFETIME=120

# Mail
MAIL_MAILER=smtp
MAIL_HOST=mailpit
MAIL_PORT=1025
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
MAIL_FROM_ADDRESS="hello@example.com"
MAIL_FROM_NAME="${APP_NAME}"

# AWS S3 (File Storage)
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=
AWS_USE_PATH_STYLE_ENDPOINT=false

# Pusher (Broadcasting)
PUSHER_APP_ID=
PUSHER_APP_KEY=
PUSHER_APP_SECRET=
PUSHER_HOST=
PUSHER_PORT=443
PUSHER_SCHEME=https
PUSHER_APP_CLUSTER=mt1
```

### สร้าง APP_KEY

```bash
# สร้าง application key (ต้องทำเสมอสำหรับ project ใหม่)
php artisan key:generate

# ผลลัพธ์:
# INFO  Application key set successfully.
```

### อ่าน Environment Variables ใน Code

```php
// ใน PHP code
$appName = env('APP_NAME', 'Default App Name');
$debug = env('APP_DEBUG', false);
$dbHost = env('DB_HOST', '127.0.0.1');

// ใน config files (แนะนำมากกว่า)
// config/app.php
return [
    'name' => env('APP_NAME', 'Laravel'),
];

// แล้วใช้ใน code ผ่าน config()
$appName = config('app.name');
$dbHost = config('database.connections.mysql.host');
```

### ข้อควรระวัง .env

```bash
# อย่า commit .env ลง git!
# ตรวจสอบว่า .gitignore มี .env
cat .gitignore | grep "^.env"
# ควรเห็น: .env

# ให้ commit .env.example แทน (ไม่มี values จริง)
# .env.example
APP_NAME=Laravel
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=
```

---

## 6. Artisan CLI

Artisan คือ command-line interface ของ Laravel

### ดู commands ทั้งหมด

```bash
php artisan list
# หรือ
php artisan --help
```

### Commands ที่ใช้บ่อย

```bash
# ===== Server =====
# รัน development server
php artisan serve
# รัน port อื่น
php artisan serve --port=8080
# รันบน host เฉพาะ
php artisan serve --host=0.0.0.0 --port=8000

# ===== Application =====
# สร้าง application key
php artisan key:generate

# ล้าง cache
php artisan cache:clear
php artisan config:clear
php artisan route:clear
php artisan view:clear

# Cache config (production)
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan optimize

# ===== Make (สร้างไฟล์) =====
# สร้าง Controller
php artisan make:controller PostController
php artisan make:controller PostController --resource  # Resource Controller
php artisan make:controller PostController --api       # API Controller

# สร้าง Model
php artisan make:model Post
php artisan make:model Post -m    # พร้อม Migration
php artisan make:model Post -mfc  # พร้อม Migration, Factory, Controller

# สร้าง Migration
php artisan make:migration create_posts_table
php artisan make:migration add_title_to_posts_table --table=posts

# สร้าง Seeder
php artisan make:seeder PostSeeder

# สร้าง Factory
php artisan make:factory PostFactory

# สร้าง Middleware
php artisan make:middleware CheckAge

# สร้าง Request
php artisan make:request StorePostRequest

# สร้าง Command
php artisan make:command SendEmails

# ===== Database =====
# รัน migrations
php artisan migrate

# Rollback migration
php artisan migrate:rollback
php artisan migrate:rollback --step=3  # Rollback 3 migrations

# Reset ทั้งหมด
php artisan migrate:reset

# Fresh (drop all tables และ migrate ใหม่)
php artisan migrate:fresh
php artisan migrate:fresh --seed  # พร้อม seed data

# รัน Seeders
php artisan db:seed
php artisan db:seed --class=PostSeeder

# ===== Routes =====
# ดู routes ทั้งหมด
php artisan route:list
php artisan route:list --name=post  # Filter by name
php artisan route:list --path=api   # Filter by path

# ===== Tinker (REPL) =====
php artisan tinker
# ใน tinker:
# >>> App\Models\User::all()
# >>> App\Models\User::count()
# >>> $user = new App\Models\User(['name' => 'John'])
```

### สร้าง Custom Artisan Command

```bash
php artisan make:command GenerateReport
```

```php
<?php
// app/Console/Commands/GenerateReport.php

namespace App\Console\Commands;

use Illuminate\Console\Command;

class GenerateReport extends Command
{
    /**
     * ชื่อและ signature ของ command
     * {month?} = optional argument
     * {--format=pdf} = option with default value
     */
    protected $signature = 'report:generate 
                            {month? : เดือนที่ต้องการ (1-12)} 
                            {--format=pdf : รูปแบบไฟล์ (pdf/excel)}
                            {--email : ส่งรายงานทาง email}';

    /**
     * คำอธิบาย command
     */
    protected $description = 'สร้างรายงานประจำเดือน';

    /**
     * Execute the console command.
     */
    public function handle(): int
    {
        $month = $this->argument('month') ?? now()->month;
        $format = $this->option('format');
        $sendEmail = $this->option('email');

        $this->info("กำลังสร้างรายงานเดือน {$month}...");

        // Progress bar
        $this->output->progressStart(100);
        for ($i = 0; $i < 100; $i++) {
            // ทำงาน...
            $this->output->progressAdvance();
        }
        $this->output->progressFinish();

        $this->info("สร้างรายงานเสร็จสิ้น! รูปแบบ: {$format}");

        if ($sendEmail) {
            $this->info("ส่ง email แล้ว");
        }

        // ถามผู้ใช้
        if ($this->confirm('ต้องการ export ไปที่ S3 ด้วยไหม?')) {
            $this->call('export:s3', ['file' => "report-{$month}.{$format}"]);
        }

        return Command::SUCCESS;
    }
}
```

```bash
# รัน command
php artisan report:generate
php artisan report:generate 3
php artisan report:generate 3 --format=excel
php artisan report:generate 3 --format=excel --email
```

### Schedule Commands (Cron)

```php
// app/Console/Kernel.php
protected function schedule(Schedule $schedule): void
{
    // รัน command ทุกวัน
    $schedule->command('report:generate')->daily();
    
    // รันทุกชั่วโมง
    $schedule->command('emails:send')->hourly();
    
    // รันทุกนาที
    $schedule->command('backup:run')->everyMinute();
    
    // รันทุกวันจันทร์-ศุกร์ เวลา 8:00
    $schedule->command('report:weekly')->weekdays()->at('08:00');
}
```

```bash
# เพิ่ม cron job บน server
# รัน: crontab -e
* * * * * cd /var/www/my-blog && php artisan schedule:run >> /dev/null 2>&1
```

---

## 7. Database Setup

### สร้าง Database

```sql
-- MySQL
CREATE DATABASE my_blog CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'laravel'@'localhost' IDENTIFIED BY 'secret';
GRANT ALL PRIVILEGES ON my_blog.* TO 'laravel'@'localhost';
FLUSH PRIVILEGES;
```

```bash
# SQLite (ง่ายที่สุดสำหรับ development)
touch database/database.sqlite

# อัปเดต .env
DB_CONNECTION=sqlite
# DB_HOST, DB_PORT, DB_DATABASE, DB_USERNAME, DB_PASSWORD ไม่จำเป็น
```

### ทดสอบ Database Connection

```bash
php artisan tinker
>>> DB::connection()->getPdo()
# ถ้าไม่ error แสดงว่า connect ได้
```

---

## 8. Laravel Development Tools

### Laravel Debugbar

```bash
composer require barryvdh/laravel-debugbar --dev
```

เปิดขึ้นมาเองเมื่อ `APP_DEBUG=true`

### Laravel Telescope

```bash
composer require laravel/telescope --dev
php artisan telescope:install
php artisan migrate
```

เข้าถึงได้ที่ `/telescope`

### IDE Helper (สำหรับ PHPStorm/VSCode)

```bash
composer require --dev barryvdh/laravel-ide-helper
php artisan ide-helper:generate
php artisan ide-helper:models
php artisan ide-helper:meta
```

---

## Workshop: สร้าง Laravel Project แรก

### โจทย์: สร้าง Blog Application พื้นฐาน

#### ขั้นตอนที่ 1: สร้างโปรเจกต์

```bash
composer create-project laravel/laravel my-first-blog
cd my-first-blog
```

#### ขั้นตอนที่ 2: ตั้งค่า Database

```bash
# แก้ไข .env
DB_CONNECTION=sqlite
```

```bash
touch database/database.sqlite
```

#### ขั้นตอนที่ 3: สร้าง Model, Migration, Controller พร้อมกัน

```bash
php artisan make:model Post -mrc
# -m = migration
# -r = resource controller
# -c = controller (ซ้ำกับ -r แต่ไม่เป็นไร)
```

#### ขั้นตอนที่ 4: แก้ไข Migration

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
            $table->string('title');
            $table->text('content');
            $table->string('slug')->unique();
            $table->boolean('published')->default(false);
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('posts');
    }
};
```

```bash
php artisan migrate
```

#### ขั้นตอนที่ 5: แก้ไข Model

```php
// app/Models/Post.php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    use HasFactory;

    protected $fillable = ['title', 'content', 'slug', 'published'];

    protected $casts = [
        'published' => 'boolean',
    ];
}
```

#### ขั้นตอนที่ 6: เพิ่ม Routes

```php
// routes/web.php
<?php

use App\Http\Controllers\PostController;
use Illuminate\Support\Facades\Route;

Route::get('/', function () {
    return view('welcome');
});

Route::resource('posts', PostController::class);
```

#### ขั้นตอนที่ 7: สร้าง Controller Logic

```php
// app/Http/Controllers/PostController.php
<?php

namespace App\Http\Controllers;

use App\Models\Post;
use Illuminate\Http\Request;
use Illuminate\Support\Str;

class PostController extends Controller
{
    public function index()
    {
        $posts = Post::latest()->paginate(10);
        return view('posts.index', compact('posts'));
    }

    public function create()
    {
        return view('posts.create');
    }

    public function store(Request $request)
    {
        $validated = $request->validate([
            'title'   => 'required|max:255',
            'content' => 'required',
        ]);

        $validated['slug'] = Str::slug($validated['title']);

        Post::create($validated);

        return redirect()->route('posts.index')
            ->with('success', 'สร้างบทความสำเร็จ!');
    }

    public function show(Post $post)
    {
        return view('posts.show', compact('post'));
    }

    public function edit(Post $post)
    {
        return view('posts.edit', compact('post'));
    }

    public function update(Request $request, Post $post)
    {
        $validated = $request->validate([
            'title'   => 'required|max:255',
            'content' => 'required',
        ]);

        $post->update($validated);

        return redirect()->route('posts.index')
            ->with('success', 'อัปเดตบทความสำเร็จ!');
    }

    public function destroy(Post $post)
    {
        $post->delete();

        return redirect()->route('posts.index')
            ->with('success', 'ลบบทความสำเร็จ!');
    }
}
```

#### ขั้นตอนที่ 8: สร้าง Views

```bash
mkdir -p resources/views/posts
```

```html
<!-- resources/views/posts/index.blade.php -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>บทความทั้งหมด</title>
    <style>
        body { font-family: sans-serif; max-width: 800px; margin: 40px auto; padding: 0 20px; }
        .post { border: 1px solid #ddd; margin: 10px 0; padding: 15px; border-radius: 5px; }
        .btn { padding: 5px 15px; text-decoration: none; border-radius: 3px; }
        .btn-primary { background: #007bff; color: white; }
        .btn-danger { background: #dc3545; color: white; }
        .alert-success { background: #d4edda; color: #155724; padding: 10px; border-radius: 3px; margin: 10px 0; }
    </style>
</head>
<body>
    <h1>บทความทั้งหมด</h1>
    
    @if(session('success'))
        <div class="alert-success">{{ session('success') }}</div>
    @endif
    
    <a href="{{ route('posts.create') }}" class="btn btn-primary">+ สร้างบทความใหม่</a>
    
    @foreach($posts as $post)
        <div class="post">
            <h2>{{ $post->title }}</h2>
            <p>{{ Str::limit($post->content, 150) }}</p>
            <a href="{{ route('posts.show', $post) }}">อ่านต่อ</a>
            |
            <a href="{{ route('posts.edit', $post) }}">แก้ไข</a>
            |
            <form method="POST" action="{{ route('posts.destroy', $post) }}" style="display:inline">
                @csrf
                @method('DELETE')
                <button type="submit" onclick="return confirm('ต้องการลบ?')">ลบ</button>
            </form>
        </div>
    @endforeach
    
    {{ $posts->links() }}
</body>
</html>
```

#### ขั้นตอนที่ 9: ทดสอบ

```bash
php artisan serve
# เปิด http://localhost:8000/posts
```

---

## Quiz

### คำถาม

**1.** คำสั่งใดที่ใช้สร้าง Controller พร้อม Model และ Migration?

a) `php artisan make:model Post --all`
b) `php artisan make:model Post -mc`
c) `php artisan make:controller PostController -m`
d) `php artisan make:post`

**2.** ไฟล์ `.env` ทำหน้าที่อะไร?

a) กำหนด routing
b) เก็บ environment-specific configuration
c) กำหนด database schema
d) ควบคุม template rendering

**3.** คำสั่งใดที่ใช้ล้าง config cache?

a) `php artisan cache:clear`
b) `php artisan config:clear`
c) `php artisan clear:config`
d) `php artisan flush:config`

**4.** ไดเรกทอรีใดที่เป็น web root (document root) ของ Laravel?

a) `/app`
b) `/resources`
c) `/public`
d) `/storage`

**5.** คำสั่งใดที่ใช้สร้าง Application Key?

a) `php artisan key:create`
b) `php artisan app:key`
c) `php artisan key:generate`
d) `php artisan generate:key`

### เฉลย

| ข้อ | คำตอบ | อธิบาย |
|-----|-------|--------|
| 1 | **a** | `--all` สร้าง Model, Migration, Factory, Seeder, Controller ครบ |
| 2 | **b** | `.env` เก็บ settings ที่แตกต่างกันในแต่ละ environment |
| 3 | **b** | `config:clear` ล้าง config cache โดยเฉพาะ |
| 4 | **c** | `/public` เป็น document root ที่ web server ชี้มา |
| 5 | **c** | `key:generate` สร้างและบันทึก APP_KEY ใน .env |

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- ข้อกำหนดระบบและการติดตั้ง PHP Extensions
- ติดตั้ง Composer และ Laravel ผ่านวิธีต่างๆ
- โครงสร้างไดเรกทอรีของ Laravel ทุกส่วน
- การตั้งค่า `.env` อย่างปลอดภัย
- Artisan CLI commands ที่ใช้บ่อย
- Workshop: สร้าง Blog application พื้นฐาน

---

## ลิงก์ไป Part ถัดไป

➡️ **[Part 027: Laravel Routing](./part-027-laravel-routing.md)**

ใน Part ถัดไปเราจะเรียนรู้การกำหนด Routes ใน Laravel อย่างละเอียด ตั้งแต่ basic routes ไปจนถึง Resource routes และ API routes
