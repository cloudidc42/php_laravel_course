# Part 037: Laravel Queues

**ระดับ:** สูง / มืออาชีพ  
**เวลาเรียน:** 4-5 ชั่วโมง  
**ความต้องการก่อนเรียน:** Part 036, Redis พื้นฐาน

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ตั้งค่าและใช้ Queue Drivers ต่างๆ ได้
- สร้าง Jobs สำหรับ background tasks
- Dispatch Jobs แบบต่างๆ
- สร้าง Job Chains และ Batches
- จัดการ Failed Jobs
- Workshop: สร้าง Email queue และ Image processing queue

---

## 1. Queue คืออะไร?

Queue (คิว) คือ pattern สำหรับทำงานที่ใช้เวลานานในพื้นหลัง (background) เช่น:
- ส่ง email
- ประมวลผลรูปภาพ
- สร้าง PDF
- เรียก API ภายนอก
- Export ข้อมูล

แทนที่ผู้ใช้จะต้องรอ เราจะโยน task เข้าคิว แล้วผู้ใช้จะได้ response ทันที

```
User Request → Controller → Dispatch Job → Queue
                                                 ↓
                                          Queue Worker รัน Job
                                                 ↓
                                            Job สำเร็จ
```

---

## 2. Queue Drivers

### 2.1 ตั้งค่าใน .env

```env
# Queue Driver
QUEUE_CONNECTION=database   # ใช้ database
# QUEUE_CONNECTION=redis    # ใช้ Redis
# QUEUE_CONNECTION=sqs      # ใช้ Amazon SQS
# QUEUE_CONNECTION=sync     # รันทันที (สำหรับ testing/development)
```

### 2.2 Database Driver

```bash
# สร้าง jobs table
php artisan queue:table
php artisan migrate
```

```env
QUEUE_CONNECTION=database
```

เหมาะสำหรับ:
- โปรเจ็กต์ขนาดเล็ก-กลาง
- ไม่ต้องการติดตั้ง Redis
- Traffic ไม่สูงมาก

### 2.3 Redis Driver

```bash
# ติดตั้ง package
composer require predis/predis
```

```env
QUEUE_CONNECTION=redis
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
```

```php
// config/queue.php
'connections' => [
    'redis' => [
        'driver' => 'redis',
        'connection' => 'default',
        'queue' => env('REDIS_QUEUE', 'default'),
        'retry_after' => 90,
        'block_for' => null,
        'after_commit' => false,
    ],
],
```

เหมาะสำหรับ:
- โปรเจ็กต์ขนาดใหญ่
- High traffic
- ต้องการ performance สูง

### 2.4 Amazon SQS Driver

```bash
composer require aws/aws-sdk-php
```

```env
QUEUE_CONNECTION=sqs
AWS_ACCESS_KEY_ID=your-key
AWS_SECRET_ACCESS_KEY=your-secret
AWS_DEFAULT_REGION=ap-southeast-1
SQS_PREFIX=https://sqs.ap-southeast-1.amazonaws.com/your-account-id
SQS_QUEUE=your-queue-name
```

เหมาะสำหรับ:
- ระบบที่ใช้ AWS
- ต้องการ highly available queue
- Serverless architecture

---

## 3. สร้าง Jobs

### 3.1 โครงสร้างพื้นฐาน

```bash
php artisan make:job SendWelcomeEmail
```

```php
<?php
// app/Jobs/SendWelcomeEmail.php

namespace App\Jobs;

use App\Models\User;
use App\Mail\WelcomeEmail;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Mail;
use Throwable;

class SendWelcomeEmail implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    /**
     * จำนวนครั้งที่ retry เมื่อ fail
     */
    public int $tries = 3;

    /**
     * เวลา timeout (วินาที)
     */
    public int $timeout = 30;

    /**
     * รอกี่วินาทีก่อน retry
     */
    public int $backoff = 10;

    /**
     * Constructor รับ data ที่ต้องการ
     */
    public function __construct(
        public readonly User $user
    ) {}

    /**
     * Execute the job.
     */
    public function handle(): void
    {
        Mail::to($this->user->email)
            ->send(new WelcomeEmail($this->user));
    }

    /**
     * Handle a job failure.
     */
    public function failed(Throwable $exception): void
    {
        // บันทึก log หรือส่ง notification เมื่อ fail
        \Log::error('SendWelcomeEmail job failed', [
            'user_id' => $this->user->id,
            'error' => $exception->getMessage(),
        ]);
    }
}
```

### 3.2 Job ที่ซับซ้อนขึ้น

```php
<?php
// app/Jobs/ProcessImage.php

namespace App\Jobs;

use App\Models\Media;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Storage;
use Intervention\Image\Facades\Image;
use Throwable;

class ProcessImage implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $timeout = 120;

    // Backoff แบบ exponential: 5, 25, 125 วินาที
    public function backoff(): array
    {
        return [5, 25, 125];
    }

    public function __construct(
        public readonly int $mediaId,
        public readonly string $operation = 'resize',
        public readonly array $options = []
    ) {}

    public function handle(): void
    {
        $media = Media::findOrFail($this->mediaId);
        
        // ดาวน์โหลดไฟล์ต้นฉบับ
        $originalPath = Storage::path($media->original_path);
        
        if (!file_exists($originalPath)) {
            throw new \Exception("Original file not found: {$originalPath}");
        }

        $image = Image::make($originalPath);

        switch ($this->operation) {
            case 'resize':
                $width = $this->options['width'] ?? 800;
                $height = $this->options['height'] ?? null;
                $image->resize($width, $height, function ($constraint) {
                    $constraint->aspectRatio();
                    $constraint->upsize();
                });
                break;

            case 'thumbnail':
                $image->fit(150, 150);
                break;

            case 'watermark':
                $watermark = Image::make(public_path('watermark.png'));
                $image->insert($watermark, 'bottom-right', 10, 10);
                break;

            case 'convert':
                // แปลงเป็น format อื่น
                $format = $this->options['format'] ?? 'webp';
                break;
        }

        // บันทึกไฟล์ที่ประมวลผลแล้ว
        $processedPath = 'media/processed/' . basename($media->original_path);
        
        $image->save(Storage::path($processedPath), 85);

        // อัพเดต database
        $media->update([
            'processed_path' => $processedPath,
            'processed_at' => now(),
            'status' => 'processed',
            'width' => $image->width(),
            'height' => $image->height(),
        ]);
    }

    public function failed(Throwable $exception): void
    {
        // อัพเดตสถานะเมื่อ fail
        Media::find($this->mediaId)?->update([
            'status' => 'failed',
            'error_message' => $exception->getMessage(),
        ]);
    }
}
```

---

## 4. Dispatching Jobs

### 4.1 การ Dispatch แบบต่างๆ

```php
// ใน Controller
use App\Jobs\SendWelcomeEmail;
use App\Jobs\ProcessImage;

class UserController extends Controller
{
    public function store(Request $request): JsonResponse
    {
        $user = User::create($request->validated());

        // Dispatch ทันที (ไปเข้า queue default)
        SendWelcomeEmail::dispatch($user);

        // Dispatch ไปยัง queue เฉพาะ
        SendWelcomeEmail::dispatch($user)->onQueue('emails');

        // Dispatch แบบ delay
        SendWelcomeEmail::dispatch($user)->delay(now()->addMinutes(5));

        // Dispatch ไปยัง connection เฉพาะ
        SendWelcomeEmail::dispatch($user)->onConnection('redis');

        // Dispatch ด้วย helper function
        dispatch(new SendWelcomeEmail($user));

        // Dispatch โดยไม่เข้า queue (รันทันที)
        SendWelcomeEmail::dispatchSync($user);

        // Dispatch ถ้าเงื่อนไขเป็น true
        SendWelcomeEmail::dispatchIf($user->email_verified, $user);

        // Dispatch ถ้าเงื่อนไขเป็น false
        SendWelcomeEmail::dispatchUnless($user->is_suspended, $user);

        return response()->json(['message' => 'User created']);
    }
}
```

### 4.2 Dispatch After DB Transaction

```php
// ป้องกันปัญหา race condition
// Job จะ dispatch หลัง transaction commit แล้ว

DB::transaction(function () use ($user) {
    $user->save();
    $user->profile()->create([...]);
    
    SendWelcomeEmail::dispatch($user)->afterCommit();
});
```

หรือตั้งค่าใน config:
```php
// config/queue.php
'connections' => [
    'database' => [
        // ...
        'after_commit' => true,  // dispatch หลัง commit เสมอ
    ],
],
```

---

## 5. Job Chains

Job Chains คือการรัน jobs ตามลำดับ job ถัดไปจะรันก็ต่อเมื่อ job ก่อนหน้าสำเร็จ

```php
use Illuminate\Support\Facades\Bus;

// Chain ง่ายๆ
Bus::chain([
    new ProcessPayment($order),
    new UpdateInventory($order),
    new SendOrderConfirmation($order),
])->dispatch();

// Chain พร้อม queue เฉพาะ
Bus::chain([
    new ProcessPayment($order),
    new SendOrderConfirmation($order),
])
->onQueue('orders')
->onConnection('redis')
->dispatch();

// จัดการ failure
Bus::chain([
    new ProcessPayment($order),
    new SendOrderConfirmation($order),
])
->catch(function (Throwable $e) use ($order) {
    // จัดการเมื่อ chain fail
    $order->update(['status' => 'payment_failed']);
    
    \Log::error('Order chain failed', [
        'order_id' => $order->id,
        'error' => $e->getMessage(),
    ]);
})
->dispatch();
```

---

## 6. Job Batches

Batches ช่วยให้รัน jobs หลายตัวพร้อมกันและติดตาม progress

```bash
# สร้าง job_batches table
php artisan queue:batches-table
php artisan migrate
```

```php
use Illuminate\Support\Facades\Bus;
use Illuminate\Bus\Batch;
use Throwable;

class ExportController extends Controller
{
    public function export(Request $request): JsonResponse
    {
        $userIds = User::where('role', 'customer')->pluck('id');
        
        // สร้าง batch
        $batch = Bus::batch(
            $userIds->map(fn($id) => new ExportUserData($id))
        )
        ->before(function (Batch $batch) {
            // ก่อน batch เริ่ม
            \Log::info("Batch {$batch->id} starting");
        })
        ->progress(function (Batch $batch) {
            // ทุกครั้งที่ job สำเร็จ
            $percentage = $batch->progress();
            \Log::info("Batch progress: {$percentage}%");
        })
        ->then(function (Batch $batch) {
            // เมื่อทุก job สำเร็จ
            \Log::info("Batch {$batch->id} completed");
        })
        ->catch(function (Batch $batch, Throwable $e) {
            // เมื่อ job fail
            \Log::error("Batch failed: " . $e->getMessage());
        })
        ->finally(function (Batch $batch) {
            // ทำงานเสมอ (ทั้ง success และ fail)
        })
        ->name('Export User Data')
        ->allowFailures()  // อนุญาตให้ fail บ้างได้
        ->onQueue('exports')
        ->dispatch();

        return response()->json([
            'batch_id' => $batch->id,
            'message' => 'Export started',
        ]);
    }

    public function progress(string $batchId): JsonResponse
    {
        $batch = Bus::findBatch($batchId);
        
        if (!$batch) {
            return response()->json(['error' => 'Batch not found'], 404);
        }

        return response()->json([
            'id' => $batch->id,
            'total_jobs' => $batch->totalJobs,
            'pending_jobs' => $batch->pendingJobs,
            'failed_jobs' => $batch->failedJobs,
            'progress' => $batch->progress(),
            'finished' => $batch->finished(),
            'failed' => $batch->hasFailures(),
        ]);
    }
}
```

---

## 7. Failed Jobs

### 7.1 ตั้งค่า Failed Jobs Table

```bash
php artisan queue:failed-table
php artisan migrate
```

### 7.2 จัดการ Failed Jobs

```bash
# ดู failed jobs ทั้งหมด
php artisan queue:failed

# Retry job เฉพาะ
php artisan queue:retry 5

# Retry ทั้งหมด
php artisan queue:retry all

# ลบ failed job
php artisan queue:forget 5

# ลบ failed jobs ทั้งหมด
php artisan queue:flush
```

### 7.3 Failed Job Hook

```php
// app/Jobs/ProcessPayment.php

public function failed(Throwable $exception): void
{
    // แจ้งเตือน admin
    \Notification::route('mail', config('app.admin_email'))
        ->notify(new PaymentJobFailed($this->order, $exception));
    
    // อัพเดตสถานะ order
    $this->order->update(['payment_status' => 'failed']);
    
    // Log error
    \Log::channel('slack')->error('Payment job failed', [
        'order_id' => $this->order->id,
        'exception' => $exception->getMessage(),
        'trace' => $exception->getTraceAsString(),
    ]);
}
```

### 7.4 Max Exceptions

```php
class ProcessPayment implements ShouldQueue
{
    // ถ้า exception เกิดขึ้น 3 ครั้ง ให้ถือว่า fail ทันที
    public int $maxExceptions = 3;
    
    public int $tries = 10; // แต่ยังคง retry ได้ 10 ครั้งถ้า exception ไม่เกิน 3 ครั้ง
}
```

---

## 8. Queue Worker

### 8.1 รัน Worker

```bash
# รัน worker พื้นฐาน
php artisan queue:work

# รัน worker สำหรับ queue เฉพาะ
php artisan queue:work --queue=emails,default

# กำหนด connection
php artisan queue:work redis --queue=emails

# กำหนด max jobs
php artisan queue:work --max-jobs=1000

# กำหนด max time (วินาที)
php artisan queue:work --max-time=3600  # 1 ชั่วโมง

# หยุดหลังจาก job แรก
php artisan queue:work --once

# Sleep เมื่อไม่มี job
php artisan queue:work --sleep=3  # รอ 3 วินาที
```

### 8.2 Supervisor Configuration

Supervisor คือ process monitor ที่จะ restart worker อัตโนมัติเมื่อ crash:

```bash
# ติดตั้ง Supervisor
sudo apt-get install supervisor

# สร้าง config file
sudo nano /etc/supervisor/conf.d/laravel-worker.conf
```

```ini
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /home/forge/app.com/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=forge
numprocs=8
redirect_stderr=true
stdout_logfile=/home/forge/app.com/storage/logs/worker.log
stopwaitsecs=3600
```

```bash
# เริ่ม Supervisor
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start laravel-worker:*

# ดูสถานะ
sudo supervisorctl status

# Restart workers
sudo supervisorctl restart laravel-worker:*
```

---

## 9. Workshop: Email Sending Queue

### 9.1 สร้าง Job

```php
<?php
// app/Jobs/SendBulkEmail.php

namespace App\Jobs;

use App\Models\User;
use App\Mail\NewsletterEmail;
use Illuminate\Bus\Batchable;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Mail;
use Throwable;

class SendBulkEmail implements ShouldQueue
{
    use Batchable, Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $timeout = 30;
    
    public function backoff(): array
    {
        return [30, 60, 120]; // วินาที
    }

    public function __construct(
        public readonly int $userId,
        public readonly string $subject,
        public readonly string $content,
        public readonly string $campaignId
    ) {}

    public function handle(): void
    {
        // ตรวจสอบว่า batch ถูกยกเลิกหรือเปล่า
        if ($this->batch()?->cancelled()) {
            return;
        }

        $user = User::findOrFail($this->userId);
        
        // ตรวจสอบ unsubscribe
        if ($user->unsubscribed_at) {
            return; // ข้ามคนที่ unsubscribe แล้ว
        }

        Mail::to($user->email)
            ->send(new NewsletterEmail($user, $this->subject, $this->content));

        // บันทึก log
        \DB::table('email_logs')->insert([
            'user_id' => $this->userId,
            'campaign_id' => $this->campaignId,
            'email' => $user->email,
            'sent_at' => now(),
            'status' => 'sent',
        ]);
    }

    public function failed(Throwable $exception): void
    {
        \DB::table('email_logs')->insert([
            'user_id' => $this->userId,
            'campaign_id' => $this->campaignId,
            'sent_at' => now(),
            'status' => 'failed',
            'error' => $exception->getMessage(),
        ]);
    }
}
```

### 9.2 Campaign Controller

```php
<?php
// app/Http/Controllers/CampaignController.php

namespace App\Http\Controllers;

use App\Jobs\SendBulkEmail;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Bus;
use Illuminate\Bus\Batch;
use Throwable;

class CampaignController extends Controller
{
    public function send(Request $request): \Illuminate\Http\JsonResponse
    {
        $request->validate([
            'subject' => 'required|string|max:255',
            'content' => 'required|string',
            'audience' => 'required|in:all,premium,free',
        ]);

        $campaignId = uniqid('campaign_');
        
        // เลือก user ตาม audience
        $users = match($request->audience) {
            'all' => User::whereNull('unsubscribed_at')->pluck('id'),
            'premium' => User::where('plan', 'premium')->whereNull('unsubscribed_at')->pluck('id'),
            'free' => User::where('plan', 'free')->whereNull('unsubscribed_at')->pluck('id'),
        };

        // สร้าง batch jobs
        $jobs = $users->map(fn($userId) => new SendBulkEmail(
            $userId,
            $request->subject,
            $request->content,
            $campaignId
        ));

        $batch = Bus::batch($jobs->all())
            ->then(fn(Batch $batch) => \Log::info("Campaign {$campaignId} completed"))
            ->catch(fn(Batch $batch, Throwable $e) => \Log::error("Campaign failed: {$e->getMessage()}"))
            ->name("Email Campaign: {$request->subject}")
            ->allowFailures()
            ->onQueue('emails')
            ->dispatch();

        // บันทึก campaign
        \DB::table('campaigns')->insert([
            'id' => $campaignId,
            'batch_id' => $batch->id,
            'subject' => $request->subject,
            'audience' => $request->audience,
            'total_recipients' => $users->count(),
            'started_at' => now(),
        ]);

        return response()->json([
            'campaign_id' => $campaignId,
            'batch_id' => $batch->id,
            'recipients' => $users->count(),
            'message' => 'Campaign started',
        ]);
    }

    public function status(string $campaignId): \Illuminate\Http\JsonResponse
    {
        $campaign = \DB::table('campaigns')->where('id', $campaignId)->first();
        
        if (!$campaign) {
            return response()->json(['error' => 'Campaign not found'], 404);
        }

        $batch = Bus::findBatch($campaign->batch_id);

        $sent = \DB::table('email_logs')
            ->where('campaign_id', $campaignId)
            ->where('status', 'sent')
            ->count();

        $failed = \DB::table('email_logs')
            ->where('campaign_id', $campaignId)
            ->where('status', 'failed')
            ->count();

        return response()->json([
            'campaign_id' => $campaignId,
            'subject' => $campaign->subject,
            'total_recipients' => $campaign->total_recipients,
            'sent' => $sent,
            'failed' => $failed,
            'progress' => $batch?->progress() ?? 100,
            'finished' => $batch?->finished() ?? true,
        ]);
    }
}
```

---

## 10. Workshop: Image Processing Queue

```php
<?php
// app/Jobs/GenerateImageVariants.php

namespace App\Jobs;

use App\Models\Media;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Storage;
use Intervention\Image\Facades\Image;
use Throwable;

class GenerateImageVariants implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $timeout = 300;

    // กำหนด sizes ที่ต้องการ
    private array $variants = [
        'thumbnail' => [150, 150],
        'small' => [400, 300],
        'medium' => [800, 600],
        'large' => [1200, 900],
    ];

    public function __construct(
        public readonly int $mediaId
    ) {}

    public function handle(): void
    {
        $media = Media::findOrFail($this->mediaId);
        
        $media->update(['status' => 'processing']);

        $originalPath = Storage::path($media->path);
        
        if (!file_exists($originalPath)) {
            throw new \RuntimeException("Original file not found");
        }

        $generatedVariants = [];

        foreach ($this->variants as $name => [$width, $height]) {
            try {
                $image = Image::make($originalPath);
                
                // Resize แบบ crop อยู่ตรงกลาง
                $image->fit($width, $height, function ($constraint) {
                    $constraint->upsize(); // ไม่ขยายถ้าเล็กกว่า
                });

                // สร้าง path สำหรับ variant
                $dir = dirname($media->path);
                $filename = pathinfo($media->path, PATHINFO_FILENAME);
                $extension = pathinfo($media->path, PATHINFO_EXTENSION);
                $variantPath = "{$dir}/{$filename}_{$name}.{$extension}";

                // บันทึก
                $image->save(Storage::path($variantPath), 85);

                $generatedVariants[$name] = [
                    'path' => $variantPath,
                    'width' => $width,
                    'height' => $height,
                    'url' => Storage::url($variantPath),
                ];

            } catch (\Exception $e) {
                \Log::warning("Failed to generate {$name} variant", [
                    'media_id' => $this->mediaId,
                    'error' => $e->getMessage(),
                ]);
            }
        }

        // อัพเดต metadata
        $media->update([
            'status' => 'processed',
            'variants' => $generatedVariants,
            'processed_at' => now(),
        ]);
    }

    public function failed(Throwable $exception): void
    {
        Media::find($this->mediaId)?->update([
            'status' => 'failed',
            'error_message' => $exception->getMessage(),
        ]);
    }
}
```

```php
// app/Http/Controllers/UploadController.php

class UploadController extends Controller
{
    public function store(Request $request): \Illuminate\Http\JsonResponse
    {
        $request->validate([
            'image' => 'required|image|max:10240', // 10MB
        ]);

        $file = $request->file('image');
        
        // บันทึกไฟล์
        $path = $file->store('uploads/originals', 'public');
        
        // สร้าง record ใน database
        $media = Media::create([
            'user_id' => auth()->id(),
            'path' => $path,
            'original_name' => $file->getClientOriginalName(),
            'mime_type' => $file->getMimeType(),
            'size' => $file->getSize(),
            'status' => 'pending',
        ]);

        // Queue การประมวลผล
        GenerateImageVariants::dispatch($media->id)
            ->onQueue('images')
            ->delay(now()->addSeconds(2)); // รอ 2 วินาทีก่อนประมวลผล

        return response()->json([
            'id' => $media->id,
            'path' => $path,
            'url' => Storage::url($path),
            'status' => 'pending',
            'message' => 'Image uploaded and queued for processing',
        ], 201);
    }

    public function status(int $mediaId): \Illuminate\Http\JsonResponse
    {
        $media = Media::findOrFail($mediaId);
        
        return response()->json([
            'id' => $media->id,
            'status' => $media->status,
            'variants' => $media->variants,
            'processed_at' => $media->processed_at,
        ]);
    }
}
```

---

## Quiz

### คำถาม 1
Queue Driver ใดเหมาะสำหรับ production ที่ต้องการ performance สูงที่สุด?

**A)** Database  
**B)** Sync  
**C)** Redis  
**D)** File  

**เฉลย: C** - Redis มี performance สูงที่สุด เพราะเป็น in-memory datastore

---

### คำถาม 2
`Job Chains` vs `Job Batches` ต่างกันอย่างไร?

**A)** Chain รันพร้อมกัน, Batch รันตามลำดับ  
**B)** Chain รันตามลำดับ (ต้องสำเร็จทีละตัว), Batch รันพร้อมกัน  
**C)** ไม่ต่างกัน  
**D)** Batch ใช้สำหรับ email เท่านั้น  

**เฉลย: B** - Chain รันตามลำดับ (series), Batch รันพร้อมกัน (parallel)

---

### คำถาม 3
`$this->batch()?->cancelled()` ในตัวอย่าง Job ใช้ทำอะไร?

**A)** ยกเลิก batch ทั้งหมด  
**B)** ตรวจสอบว่า batch ถูกยกเลิกแล้วหรือไม่  
**C)** หยุด job ชั่วคราว  
**D)** ดูจำนวน cancelled jobs  

**เฉลย: B** - ตรวจสอบว่า batch ถูกยกเลิกก่อนรัน job ต่อไป

---

### คำถาม 4
Supervisor ทำหน้าที่อะไรในระบบ Queue?

**A)** รัน queue jobs  
**B)** Monitor และ restart queue workers อัตโนมัติเมื่อ crash  
**C)** บันทึก log ของ jobs  
**D)** จัดการ failed jobs  

**เฉลย: B** - Supervisor เป็น process monitor ที่คอย restart workers เมื่อ crash

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- ✅ Queue Drivers ต่างๆ (Database, Redis, SQS)
- ✅ การสร้างและ Dispatch Jobs
- ✅ Job Chains สำหรับ sequential tasks
- ✅ Job Batches สำหรับ parallel tasks พร้อม progress tracking
- ✅ การจัดการ Failed Jobs
- ✅ Workshop: Bulk Email Campaign และ Image Processing

---

## ลิงก์ไป Part ถัดไป

➡️ **[Part 038: Laravel Events & Broadcasting](part-038-laravel-events.md)**  
เรียนรู้เรื่อง Events, Listeners, Queued Listeners และ Real-time Broadcasting ด้วย Pusher/Soketi
