# Part 036: Laravel Artisan Commands

**ระดับ:** สูง / มืออาชีพ  
**เวลาเรียน:** 3-4 ชั่วโมง  
**ความต้องการก่อนเรียน:** ความรู้พื้นฐาน Laravel, OOP PHP

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เข้าใจและใช้ Artisan commands ทั้งหมดที่มีใน Laravel
- สร้าง Custom Artisan Command ของตัวเองได้
- ตั้งค่า Command Scheduling (Cron) สำหรับ automated tasks
- สร้าง maintenance tools สำหรับ production

---

## 1. Artisan คืออะไร?

Artisan คือ command-line interface (CLI) ที่มาพร้อมกับ Laravel ชื่อมาจาก "artisan" ซึ่งหมายถึงช่างฝีมือ เพราะ Artisan ช่วยให้นักพัฒนาสร้างสิ่งต่างๆ ได้อย่างรวดเร็วและมีประสิทธิภาพ

```bash
# ดู commands ทั้งหมด
php artisan list

# ดูความช่วยเหลือของ command
php artisan help make:model
```

---

## 2. Artisan Commands พื้นฐาน

### 2.1 Application Commands

```bash
# เริ่มต้น development server
php artisan serve
php artisan serve --port=8080

# แสดง routes ทั้งหมด
php artisan route:list
php artisan route:list --method=GET
php artisan route:list --name=api

# Clear caches
php artisan cache:clear
php artisan config:clear
php artisan route:clear
php artisan view:clear
php artisan optimize:clear

# Cache ทุกอย่าง (สำหรับ production)
php artisan optimize
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

### 2.2 Database Commands

```bash
# รัน migrations
php artisan migrate
php artisan migrate --force    # สำหรับ production
php artisan migrate:fresh      # Drop ทั้งหมดแล้ว migrate ใหม่
php artisan migrate:refresh    # Rollback แล้ว migrate ใหม่
php artisan migrate:rollback   # Rollback migration ล่าสุด
php artisan migrate:rollback --step=3  # Rollback 3 steps
php artisan migrate:status     # ดูสถานะ migration

# Seeders
php artisan db:seed
php artisan db:seed --class=UserSeeder
php artisan migrate:fresh --seed
```

### 2.3 Make Commands (Scaffolding)

```bash
# สร้าง Model พร้อม migration
php artisan make:model Product -m

# สร้าง Model พร้อมทุกอย่าง
php artisan make:model Product -mfsc
# -m = migration
# -f = factory
# -s = seeder
# -c = controller

# สร้าง Controller
php artisan make:controller ProductController
php artisan make:controller ProductController --resource  # CRUD methods
php artisan make:controller Api/ProductController --api   # API controller

# สร้าง Migration
php artisan make:migration create_products_table
php artisan make:migration add_price_to_products_table --table=products

# สร้าง Seeder
php artisan make:seeder ProductSeeder

# สร้าง Factory
php artisan make:factory ProductFactory

# สร้าง Middleware
php artisan make:middleware CheckAge

# สร้าง Request (Form Validation)
php artisan make:request StoreProductRequest

# สร้าง Policy
php artisan make:policy ProductPolicy
php artisan make:policy ProductPolicy --model=Product

# สร้าง Resource (API)
php artisan make:resource ProductResource
php artisan make:resource ProductCollection

# สร้าง Event & Listener
php artisan make:event OrderShipped
php artisan make:listener SendOrderShippedNotification --event=OrderShipped

# สร้าง Job
php artisan make:job ProcessPayment

# สร้าง Mail
php artisan make:mail OrderConfirmation
php artisan make:mail OrderConfirmation --markdown=emails.orders.confirmation

# สร้าง Notification
php artisan make:notification InvoicePaid

# สร้าง Observer
php artisan make:observer ProductObserver --model=Product

# สร้าง Provider
php artisan make:provider PaymentServiceProvider

# สร้าง Command
php artisan make:command SendEmails
```

### 2.4 Queue Commands

```bash
# รัน Queue Worker
php artisan queue:work
php artisan queue:work --queue=emails
php artisan queue:work --tries=3
php artisan queue:work --timeout=60

# รัน Queue ด้วย Supervisor (แนะนำสำหรับ production)
# ดูรายละเอียดใน part-037

# ดู Failed Jobs
php artisan queue:failed
php artisan queue:retry all
php artisan queue:flush

# Horizon (สำหรับ Redis)
php artisan horizon
php artisan horizon:pause
php artisan horizon:continue
```

### 2.5 Tinker

```bash
# เปิด REPL สำหรับทดสอบ code
php artisan tinker

# ตัวอย่างใน Tinker
>>> User::all()
>>> User::find(1)
>>> User::create(['name' => 'Test', 'email' => 'test@test.com', 'password' => bcrypt('password')])
>>> DB::table('users')->count()
```

---

## 3. สร้าง Custom Artisan Command

### 3.1 โครงสร้างพื้นฐาน

```bash
php artisan make:command SendDailyReport
```

ไฟล์จะถูกสร้างที่ `app/Console/Commands/SendDailyReport.php`:

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;

class SendDailyReport extends Command
{
    /**
     * ชื่อและ signature ของ command
     * รูปแบบ: name {argument} {--option}
     */
    protected $signature = 'report:daily';

    /**
     * คำอธิบาย command
     */
    protected $description = 'Send daily report to administrators';

    /**
     * Execute the console command.
     */
    public function handle(): int
    {
        $this->info('Sending daily report...');
        
        // Logic ของเรา
        
        $this->info('Daily report sent successfully!');
        
        return Command::SUCCESS; // หรือ Command::FAILURE
    }
}
```

### 3.2 Arguments และ Options

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use App\Models\User;
use App\Mail\DailyReport;
use Illuminate\Support\Facades\Mail;

class SendDailyReport extends Command
{
    /**
     * {recipient} = required argument
     * {--type=all} = optional option พร้อม default value
     * {--force} = boolean flag
     * {--emails=*} = array option
     */
    protected $signature = 'report:daily 
                            {recipient : The email recipient}
                            {--type=all : Type of report (all/sales/users)}
                            {--force : Force send even if already sent today}
                            {--emails=* : Additional email addresses}';

    protected $description = 'Send daily report to specified recipient';

    public function handle(): int
    {
        // รับ argument
        $recipient = $this->argument('recipient');
        
        // รับ option
        $type = $this->option('type');
        $force = $this->option('force');
        $additionalEmails = $this->option('emails');

        $this->info("Sending {$type} report to: {$recipient}");
        
        if ($force) {
            $this->warn('Force mode enabled - will send even if already sent');
        }

        // ใช้ progress bar
        $users = User::all();
        $bar = $this->output->createProgressBar(count($users));
        $bar->start();

        foreach ($users as $user) {
            // ทำงานกับ user
            sleep(0.01); // จำลอง
            $bar->advance();
        }

        $bar->finish();
        $this->newLine();
        
        // แสดงตาราง
        $this->table(
            ['Name', 'Email', 'Role'],
            User::select('name', 'email', 'role')->take(5)->get()->toArray()
        );

        // ถามคำถาม
        if ($this->confirm('Do you want to send test email?')) {
            $this->info('Sending test email...');
        }

        // เลือกจาก list
        $queue = $this->choice(
            'Which queue to use?',
            ['default', 'emails', 'reports'],
            0 // default index
        );

        $this->info("Using queue: {$queue}");
        
        return Command::SUCCESS;
    }
}
```

### 3.3 ตัวอย่างจริง: Database Cleanup Command

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use App\Models\User;
use App\Models\AuditLog;
use App\Models\TempFile;
use Carbon\Carbon;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Facades\DB;

class CleanupDatabase extends Command
{
    protected $signature = 'db:cleanup 
                            {--days=30 : Delete records older than X days}
                            {--dry-run : Show what would be deleted without deleting}
                            {--model=all : Which model to cleanup (all/logs/files/sessions)}';

    protected $description = 'Clean up old records from database';

    public function handle(): int
    {
        $days = (int) $this->option('days');
        $dryRun = $this->option('dry-run');
        $model = $this->option('model');
        
        $cutoffDate = Carbon::now()->subDays($days);
        
        $this->info("Cleanup started - removing records older than {$days} days");
        $this->info("Cutoff date: " . $cutoffDate->format('Y-m-d H:i:s'));
        
        if ($dryRun) {
            $this->warn('DRY RUN MODE - No actual deletions will occur');
        }

        $totalDeleted = 0;

        // Cleanup Audit Logs
        if ($model === 'all' || $model === 'logs') {
            $count = $this->cleanupAuditLogs($cutoffDate, $dryRun);
            $totalDeleted += $count;
            $this->line("  Audit logs: {$count} records");
        }

        // Cleanup Temp Files
        if ($model === 'all' || $model === 'files') {
            $count = $this->cleanupTempFiles($cutoffDate, $dryRun);
            $totalDeleted += $count;
            $this->line("  Temp files: {$count} records");
        }

        // Cleanup Sessions
        if ($model === 'all' || $model === 'sessions') {
            $count = $this->cleanupSessions($cutoffDate, $dryRun);
            $totalDeleted += $count;
            $this->line("  Sessions: {$count} records");
        }

        $this->newLine();
        
        if ($dryRun) {
            $this->info("Would delete {$totalDeleted} records total (dry run)");
        } else {
            $this->info("Successfully deleted {$totalDeleted} records total");
        }

        return Command::SUCCESS;
    }

    private function cleanupAuditLogs(Carbon $cutoffDate, bool $dryRun): int
    {
        $query = AuditLog::where('created_at', '<', $cutoffDate);
        $count = $query->count();
        
        if (!$dryRun && $count > 0) {
            $query->delete();
        }
        
        return $count;
    }

    private function cleanupTempFiles(Carbon $cutoffDate, bool $dryRun): int
    {
        $files = TempFile::where('created_at', '<', $cutoffDate)->get();
        $count = $files->count();
        
        if (!$dryRun && $count > 0) {
            foreach ($files as $file) {
                // ลบไฟล์จาก storage ด้วย
                if (Storage::exists($file->path)) {
                    Storage::delete($file->path);
                }
            }
            
            TempFile::where('created_at', '<', $cutoffDate)->delete();
        }
        
        return $count;
    }

    private function cleanupSessions(Carbon $cutoffDate, bool $dryRun): int
    {
        $query = DB::table('sessions')
            ->where('last_activity', '<', $cutoffDate->timestamp);
        
        $count = $query->count();
        
        if (!$dryRun && $count > 0) {
            $query->delete();
        }
        
        return $count;
    }
}
```

```bash
# ใช้งาน command
php artisan db:cleanup
php artisan db:cleanup --days=7
php artisan db:cleanup --dry-run
php artisan db:cleanup --model=logs --days=90
```

### 3.4 Command ที่เรียก Command อื่น

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;

class MaintenanceMode extends Command
{
    protected $signature = 'app:maintenance 
                            {action : Action to perform (start/end)}
                            {--message=We are performing maintenance : Maintenance message}';

    protected $description = 'Put application in maintenance mode';

    public function handle(): int
    {
        $action = $this->argument('action');
        $message = $this->option('message');

        if ($action === 'start') {
            $this->startMaintenance($message);
        } elseif ($action === 'end') {
            $this->endMaintenance();
        } else {
            $this->error("Invalid action. Use 'start' or 'end'");
            return Command::FAILURE;
        }

        return Command::SUCCESS;
    }

    private function startMaintenance(string $message): void
    {
        $this->info('Starting maintenance mode...');
        
        // เรียก built-in artisan command
        $this->call('down', [
            '--message' => $message,
            '--retry' => 60,
        ]);
        
        // Clear caches
        $this->call('cache:clear');
        $this->call('config:clear');
        
        $this->info('Application is now in maintenance mode');
    }

    private function endMaintenance(): void
    {
        $this->info('Ending maintenance mode...');
        
        // Warm up caches
        $this->call('config:cache');
        $this->call('route:cache');
        $this->call('view:cache');
        
        // Bring app back up
        $this->call('up');
        
        $this->info('Application is back online!');
    }
}
```

---

## 4. Command Scheduling (Cron)

### 4.1 การตั้งค่า Kernel

ใน Laravel 10 และก่อนหน้า ใช้ `app/Console/Kernel.php`:

```php
<?php

namespace App\Console;

use Illuminate\Console\Scheduling\Schedule;
use Illuminate\Foundation\Console\Kernel as ConsoleKernel;
use App\Console\Commands\SendDailyReport;
use App\Console\Commands\CleanupDatabase;

class Kernel extends ConsoleKernel
{
    /**
     * Define the application's command schedule.
     */
    protected function schedule(Schedule $schedule): void
    {
        // รัน command ทุกวันตอนเที่ยงคืน
        $schedule->command('report:daily admin@company.com')
                 ->dailyAt('00:00')
                 ->emailOutputTo('admin@company.com');

        // รันทุกชั่วโมง
        $schedule->command('db:cleanup --days=30')
                 ->hourly()
                 ->withoutOverlapping();  // ป้องกัน overlap

        // รันทุกวันจันทร์
        $schedule->command('report:weekly')
                 ->weekly()
                 ->mondays()
                 ->at('09:00');

        // รันทุกวันแรกของเดือน
        $schedule->command('report:monthly')
                 ->monthlyOn(1, '06:00');

        // เรียก Closure
        $schedule->call(function () {
            \DB::table('temp_tokens')
               ->where('expires_at', '<', now())
               ->delete();
        })->everyFiveMinutes();

        // รัน shell command
        $schedule->exec('node /path/to/script.js')
                 ->daily();
        
        // รันเฉพาะ environment
        $schedule->command('report:daily admin@company.com')
                 ->daily()
                 ->environments(['production']);
    }

    /**
     * Register the commands for the application.
     */
    protected function commands(): void
    {
        $this->load(__DIR__.'/Commands');
        require base_path('routes/console.php');
    }
}
```

### 4.2 ใน Laravel 11+ (ใช้ routes/console.php)

```php
<?php
// routes/console.php

use Illuminate\Support\Facades\Schedule;

// รัน artisan command
Schedule::command('report:daily admin@company.com')->daily();

// รัน closure
Schedule::call(function () {
    \DB::table('sessions')->where('last_activity', '<', now()->subHours(2))->delete();
})->everyFifteenMinutes();
```

### 4.3 Frequency Options ทั้งหมด

```php
// ทุกนาที
$schedule->command('task')->everyMinute();
$schedule->command('task')->everyTwoMinutes();
$schedule->command('task')->everyFiveMinutes();
$schedule->command('task')->everyTenMinutes();
$schedule->command('task')->everyFifteenMinutes();
$schedule->command('task')->everyThirtyMinutes();

// ทุกชั่วโมง
$schedule->command('task')->hourly();
$schedule->command('task')->hourlyAt(17);   // ทุกชั่วโมงที่นาทีที่ 17
$schedule->command('task')->everyOddHour(); // ทุกชั่วโมงคี่
$schedule->command('task')->everyTwoHours();

// ทุกวัน
$schedule->command('task')->daily();
$schedule->command('task')->dailyAt('13:00');
$schedule->command('task')->twiceDaily(1, 13); // 01:00 และ 13:00

// ทุกสัปดาห์
$schedule->command('task')->weekly();
$schedule->command('task')->weeklyOn(1, '8:00'); // จันทร์ 08:00

// ทุกเดือน
$schedule->command('task')->monthly();
$schedule->command('task')->monthlyOn(4, '15:00'); // วันที่ 4 เวลา 15:00
$schedule->command('task')->lastDayOfMonth('15:00');

// วันในสัปดาห์
$schedule->command('task')->weekdays();    // จันทร์-ศุกร์
$schedule->command('task')->weekends();   // เสาร์-อาทิตย์
$schedule->command('task')->mondays();
$schedule->command('task')->tuesdays();
// ... etc

// Custom cron expression
$schedule->command('task')->cron('0 9 * * 1-5'); // จ-ศ เวลา 9:00
```

### 4.4 ตั้งค่า Cron บน Server

```bash
# เปิด crontab
crontab -e

# เพิ่ม cron job (รันทุกนาที)
* * * * * cd /path-to-your-project && php artisan schedule:run >> /dev/null 2>&1

# สำหรับ Forge/Envoyer
* * * * * forge php /home/forge/yoursite.com/artisan schedule:run >> /dev/null 2>&1
```

### 4.5 Schedule Output Handling

```php
$schedule->command('emails:send')
         ->daily()
         ->sendOutputTo('/var/log/artisan/emails.log')
         ->appendOutputTo('/var/log/artisan/emails.log')  // append
         ->emailOutputTo('admin@example.com')             // email
         ->emailOutputOnFailure('admin@example.com');     // email เฉพาะเมื่อ fail
```

### 4.6 Hooks

```php
$schedule->command('db:cleanup')
         ->daily()
         ->before(function () {
             // ก่อนรัน
             \Log::info('Cleanup starting...');
         })
         ->after(function () {
             // หลังรัน
             \Log::info('Cleanup completed');
         })
         ->onSuccess(function () {
             // เมื่อสำเร็จ
         })
         ->onFailure(function () {
             // เมื่อล้มเหลว
             \Notification::route('slack', config('logging.channels.slack.url'))
                          ->notify(new ScheduledTaskFailed('db:cleanup'));
         });
```

---

## 5. Workshop: สร้าง Maintenance Tools

### Workshop 1: Database Health Check Command

```php
<?php
// app/Console/Commands/DatabaseHealthCheck.php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Cache;

class DatabaseHealthCheck extends Command
{
    protected $signature = 'db:health 
                            {--fix : Try to fix issues automatically}
                            {--notify : Send notification if issues found}';

    protected $description = 'Check database health and performance';

    public function handle(): int
    {
        $this->info('=== Database Health Check ===');
        $this->newLine();

        $issues = [];

        // Check 1: Connection
        $this->line('1. Checking database connection...');
        if (!$this->checkConnection()) {
            $this->error('   ✗ Cannot connect to database!');
            return Command::FAILURE;
        }
        $this->info('   ✓ Database connection OK');

        // Check 2: Table counts
        $this->line('2. Checking table statistics...');
        $stats = $this->getTableStats();
        $this->table(
            ['Table', 'Rows', 'Size (MB)'],
            $stats
        );

        // Check 3: Large tables
        $this->line('3. Checking for large tables...');
        $largeTables = array_filter($stats, fn($s) => $s[2] > 100);
        if (!empty($largeTables)) {
            $this->warn('   ⚠ Found large tables (>100MB):');
            foreach ($largeTables as $table) {
                $this->line("     - {$table[0]}: {$table[2]}MB");
            }
            $issues[] = 'Large tables found';
        } else {
            $this->info('   ✓ No oversized tables');
        }

        // Check 4: Orphan records
        $this->line('4. Checking for orphan records...');
        $orphans = $this->checkOrphanRecords();
        if ($orphans > 0) {
            $this->warn("   ⚠ Found {$orphans} orphan records");
            $issues[] = "Orphan records: {$orphans}";
            
            if ($this->option('fix')) {
                $this->line('   Fixing orphan records...');
                $this->fixOrphanRecords();
                $this->info('   ✓ Orphan records fixed');
            }
        } else {
            $this->info('   ✓ No orphan records');
        }

        // Check 5: Cache test
        $this->line('5. Testing cache...');
        if ($this->testCache()) {
            $this->info('   ✓ Cache is working');
        } else {
            $this->error('   ✗ Cache is not working!');
            $issues[] = 'Cache not working';
        }

        $this->newLine();

        if (empty($issues)) {
            $this->info('✓ All checks passed! Database is healthy.');
        } else {
            $this->warn('⚠ Found ' . count($issues) . ' issue(s):');
            foreach ($issues as $issue) {
                $this->line("  - {$issue}");
            }

            if ($this->option('notify')) {
                $this->notifyAdmins($issues);
                $this->info('Admins have been notified');
            }

            return Command::FAILURE;
        }

        return Command::SUCCESS;
    }

    private function checkConnection(): bool
    {
        try {
            DB::connection()->getPdo();
            return true;
        } catch (\Exception $e) {
            return false;
        }
    }

    private function getTableStats(): array
    {
        return DB::select("
            SELECT 
                table_name,
                table_rows,
                ROUND(((data_length + index_length) / 1024 / 1024), 2) AS size_mb
            FROM information_schema.TABLES
            WHERE table_schema = ?
            ORDER BY (data_length + index_length) DESC
            LIMIT 20
        ", [config('database.connections.mysql.database')]);
    }

    private function checkOrphanRecords(): int
    {
        // ตรวจสอบ comments ที่ไม่มี post
        return DB::table('comments')
            ->whereNotExists(function ($query) {
                $query->select(DB::raw(1))
                      ->from('posts')
                      ->whereColumn('posts.id', 'comments.post_id');
            })
            ->count();
    }

    private function fixOrphanRecords(): void
    {
        DB::table('comments')
            ->whereNotExists(function ($query) {
                $query->select(DB::raw(1))
                      ->from('posts')
                      ->whereColumn('posts.id', 'comments.post_id');
            })
            ->delete();
    }

    private function testCache(): bool
    {
        $key = 'health_check_' . time();
        Cache::put($key, 'test', 10);
        $result = Cache::get($key) === 'test';
        Cache::forget($key);
        return $result;
    }

    private function notifyAdmins(array $issues): void
    {
        // Send notification
        \Notification::route('mail', config('app.admin_email'))
            ->notify(new \App\Notifications\DatabaseIssuesFound($issues));
    }
}
```

### Workshop 2: Generate Sitemap Command

```php
<?php
// app/Console/Commands/GenerateSitemap.php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use App\Models\Post;
use App\Models\Category;
use Carbon\Carbon;

class GenerateSitemap extends Command
{
    protected $signature = 'sitemap:generate 
                            {--output=public/sitemap.xml : Output file path}';

    protected $description = 'Generate XML sitemap for the website';

    public function handle(): int
    {
        $output = $this->option('output');
        
        $this->info('Generating sitemap...');

        $xml = $this->buildSitemapXml();
        
        file_put_contents(base_path($output), $xml);
        
        $this->info("Sitemap generated: {$output}");
        $this->line('Total URLs: ' . substr_count($xml, '<url>'));

        return Command::SUCCESS;
    }

    private function buildSitemapXml(): string
    {
        $urls = collect();

        // หน้าหลัก
        $urls->push([
            'loc' => url('/'),
            'lastmod' => now()->format('Y-m-d'),
            'changefreq' => 'daily',
            'priority' => '1.0',
        ]);

        // Posts
        Post::published()
            ->select('slug', 'updated_at')
            ->chunk(100, function ($posts) use (&$urls) {
                foreach ($posts as $post) {
                    $urls->push([
                        'loc' => route('posts.show', $post->slug),
                        'lastmod' => $post->updated_at->format('Y-m-d'),
                        'changefreq' => 'weekly',
                        'priority' => '0.8',
                    ]);
                }
            });

        // Categories
        Category::all()->each(function ($category) use (&$urls) {
            $urls->push([
                'loc' => route('categories.show', $category->slug),
                'lastmod' => now()->format('Y-m-d'),
                'changefreq' => 'daily',
                'priority' => '0.6',
            ]);
        });

        // Generate XML
        $xml = '<?xml version="1.0" encoding="UTF-8"?>' . PHP_EOL;
        $xml .= '<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">' . PHP_EOL;

        foreach ($urls as $url) {
            $xml .= '  <url>' . PHP_EOL;
            $xml .= '    <loc>' . htmlspecialchars($url['loc']) . '</loc>' . PHP_EOL;
            $xml .= '    <lastmod>' . $url['lastmod'] . '</lastmod>' . PHP_EOL;
            $xml .= '    <changefreq>' . $url['changefreq'] . '</changefreq>' . PHP_EOL;
            $xml .= '    <priority>' . $url['priority'] . '</priority>' . PHP_EOL;
            $xml .= '  </url>' . PHP_EOL;
        }

        $xml .= '</urlset>';

        return $xml;
    }
}
```

### Schedule ทั้งหมด

```php
// app/Console/Kernel.php หรือ routes/console.php

// Health check ทุก 6 ชั่วโมง
$schedule->command('db:health --notify')
         ->everySixHours()
         ->withoutOverlapping()
         ->onFailure(function () {
             \Log::critical('Database health check failed!');
         });

// Generate sitemap ทุกวัน
$schedule->command('sitemap:generate')
         ->dailyAt('03:00')
         ->runInBackground();  // รันใน background

// Cleanup เก่า
$schedule->command('db:cleanup --days=90')
         ->weekly()
         ->sundays()
         ->at('02:00');
```

---

## Quiz

### คำถาม 1
คำสั่ง `php artisan make:model Product -mfsc` จะสร้างอะไรบ้าง?

**A)** Model อย่างเดียว  
**B)** Model + Migration + Factory + Seeder + Controller  
**C)** Model + Migration + Factory  
**D)** Model + Controller  

**เฉลย: B** - flags `-m` (migration), `-f` (factory), `-s` (seeder), `-c` (controller)

---

### คำถาม 2
ถ้าต้องการรัน scheduled task ทุกวันจันทร์-ศุกร์ เวลา 9:00 น. ควรใช้ cron expression อะไร?

**A)** `0 9 * * *`  
**B)** `0 9 * * 1-5`  
**C)** `9 0 * * 1-5`  
**D)** `* 9 1-5 * *`  

**เฉลย: B** - `0 9 * * 1-5` หมายถึง นาทีที่ 0, ชั่วโมง 9, ทุกวัน, ทุกเดือน, วันจันทร์-ศุกร์ (1-5)

---

### คำถาม 3
Method ใดใน Command class ที่ใช้แสดง progress bar?

**A)** `$this->showProgress()`  
**B)** `$this->progress()`  
**C)** `$this->output->createProgressBar()`  
**D)** `$this->progressBar()`  

**เฉลย: C** - `$this->output->createProgressBar($count)` สร้าง progress bar

---

### คำถาม 4
`withoutOverlapping()` ใน scheduler ทำหน้าที่อะไร?

**A)** ป้องกันไม่ให้ command รันซ้อนกัน  
**B)** ทำให้ command รันเร็วขึ้น  
**C)** ป้องกัน memory leak  
**D)** ทำให้ output ไม่แสดง  

**เฉลย: A** - ป้องกันไม่ให้ command รันขณะที่ instance ก่อนหน้ายังทำงานอยู่

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- ✅ Artisan commands พื้นฐานทั้งหมด
- ✅ การสร้าง Custom Artisan Command
- ✅ การใช้ Arguments, Options และ Output formatting
- ✅ Command Scheduling และ Cron
- ✅ Workshop: Database Health Check และ Sitemap Generator

---

## ลิงก์ไป Part ถัดไป

➡️ **[Part 037: Laravel Queues](part-037-laravel-queues.md)**  
เรียนรู้เรื่อง Queue Drivers, Jobs, Dispatching และ Failed Jobs เพื่อทำงาน background tasks อย่างมีประสิทธิภาพ
