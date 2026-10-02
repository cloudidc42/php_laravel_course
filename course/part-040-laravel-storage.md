# Part 040: Laravel Storage

**ระดับ:** สูง / มืออาชีพ  
**เวลาเรียน:** 3-4 ชั่วโมง  
**ความต้องการก่อนเรียน:** Part 037 (Queues), PHP File I/O พื้นฐาน

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ Storage facade กับ drivers ต่างๆ ได้
- Upload และประมวลผลรูปภาพ
- จัดการ File Visibility (public/private)
- สร้าง Custom Filesystem Driver
- Workshop: Profile picture upload และ Document management

---

## 1. Filesystem Configuration

### 1.1 config/filesystems.php

```php
<?php

return [
    'default' => env('FILESYSTEM_DISK', 'local'),

    'disks' => [
        // Local storage (ไฟล์เฉพาะ server)
        'local' => [
            'driver' => 'local',
            'root' => storage_path('app'),
            'throw' => false,
        ],

        // Public storage (เข้าถึงได้จาก browser)
        'public' => [
            'driver' => 'local',
            'root' => storage_path('app/public'),
            'url' => env('APP_URL') . '/storage',
            'visibility' => 'public',
            'throw' => false,
        ],

        // Amazon S3
        's3' => [
            'driver' => 's3',
            'key' => env('AWS_ACCESS_KEY_ID'),
            'secret' => env('AWS_SECRET_ACCESS_KEY'),
            'region' => env('AWS_DEFAULT_REGION'),
            'bucket' => env('AWS_BUCKET'),
            'url' => env('AWS_URL'),
            'endpoint' => env('AWS_ENDPOINT'),
            'use_path_style_endpoint' => env('AWS_USE_PATH_STYLE_ENDPOINT', false),
            'throw' => false,
        ],

        // MinIO (S3-compatible, self-hosted)
        'minio' => [
            'driver' => 's3',
            'key' => env('MINIO_KEY', 'minioadmin'),
            'secret' => env('MINIO_SECRET', 'minioadmin'),
            'region' => env('MINIO_REGION', 'us-east-1'),
            'bucket' => env('MINIO_BUCKET', 'myapp'),
            'url' => env('MINIO_URL', 'http://localhost:9000'),
            'endpoint' => env('MINIO_ENDPOINT', 'http://localhost:9000'),
            'use_path_style_endpoint' => true,
        ],

        // Google Cloud Storage
        'gcs' => [
            'driver' => 'gcs',
            'key_file_path' => env('GOOGLE_CLOUD_KEY_FILE', null),
            'project_id' => env('GOOGLE_CLOUD_PROJECT_ID'),
            'bucket' => env('GOOGLE_CLOUD_STORAGE_BUCKET'),
            'path_prefix' => env('GOOGLE_CLOUD_STORAGE_PATH_PREFIX', null),
        ],

        // เพิ่ม disk สำหรับ backups
        'backups' => [
            'driver' => 'local',
            'root' => storage_path('backups'),
        ],
    ],
];
```

### 1.2 Symbolic Link

```bash
# สร้าง symbolic link จาก public/storage ไป storage/app/public
php artisan storage:link

# ตรวจสอบ
ls -la public/storage
```

---

## 2. Storage Facade

### 2.1 Operations พื้นฐาน

```php
use Illuminate\Support\Facades\Storage;

// ===== อ่านเขียนไฟล์ =====

// เขียนไฟล์
Storage::put('file.txt', 'Contents');
Storage::disk('local')->put('file.txt', 'Contents');
Storage::disk('s3')->put('file.txt', 'Contents');

// เขียนแบบ prepend/append
Storage::prepend('file.txt', 'Prepended text');
Storage::append('file.txt', 'Appended text');

// อ่านไฟล์
$contents = Storage::get('file.txt');

// อ่านทีละ chunk (ไฟล์ใหญ่)
Storage::readStream('large-file.csv');

// ===== ตรวจสอบ =====

Storage::exists('file.txt');     // ไฟล์มีหรือเปล่า
Storage::missing('file.txt');    // ไฟล์ไม่มีหรือเปล่า
Storage::size('file.txt');       // ขนาดไฟล์ (bytes)
Storage::lastModified('file.txt'); // เวลาแก้ไขล่าสุด

// ===== ลบ =====

Storage::delete('file.txt');
Storage::delete(['file1.txt', 'file2.txt']);

// ===== ย้ายและคัดลอก =====

Storage::copy('old/file.txt', 'new/file.txt');
Storage::move('old/file.txt', 'new/file.txt');

// ===== Directory =====

Storage::files('path');          // ไฟล์ใน directory
Storage::allFiles('path');       // ไฟล์ทั้งหมด (รวม subdirectory)
Storage::directories('path');   // directories ใน path
Storage::makeDirectory('path');
Storage::deleteDirectory('path');

// ===== URL =====

Storage::url('file.txt');              // URL สำหรับ public disk
Storage::temporaryUrl('file.txt', now()->addMinutes(30)); // Signed URL (S3)
```

---

## 3. File Upload

### 3.1 Basic Upload

```php
<?php
// app/Http/Controllers/UploadController.php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;

class UploadController extends Controller
{
    public function upload(Request $request): \Illuminate\Http\JsonResponse
    {
        $request->validate([
            'file' => 'required|file|max:10240', // 10MB
        ]);

        $file = $request->file('file');

        // วิธีที่ 1: Laravel จัดการชื่อให้อัตโนมัติ (UUID)
        $path = $file->store('uploads', 'public');

        // วิธีที่ 2: กำหนดชื่อเอง
        $filename = Str::uuid() . '.' . $file->getClientOriginalExtension();
        $path = $file->storeAs('uploads', $filename, 'public');

        // วิธีที่ 3: ใช้ Storage::put โดยตรง
        $path = 'uploads/' . $filename;
        Storage::disk('public')->put($path, file_get_contents($file));

        return response()->json([
            'path' => $path,
            'url' => Storage::disk('public')->url($path),
            'size' => $file->getSize(),
            'mime' => $file->getMimeType(),
        ]);
    }
}
```

### 3.2 Image Upload พร้อม Validation

```php
public function uploadImage(Request $request): \Illuminate\Http\JsonResponse
{
    $request->validate([
        'image' => [
            'required',
            'image',
            'mimes:jpeg,jpg,png,gif,webp',
            'max:5120',  // 5MB
            'dimensions:min_width=100,min_height=100,max_width=5000,max_height=5000',
        ],
    ]);

    $file = $request->file('image');
    
    // ตรวจสอบ MIME type จริงๆ (ไม่เชื่อ extension อย่างเดียว)
    $finfo = new \finfo(FILEINFO_MIME_TYPE);
    $mimeType = $finfo->file($file->getPathname());
    
    $allowedMimes = ['image/jpeg', 'image/png', 'image/gif', 'image/webp'];
    if (!in_array($mimeType, $allowedMimes)) {
        return response()->json(['error' => 'Invalid file type'], 422);
    }

    // สร้างชื่อไฟล์ที่ปลอดภัย
    $extension = match($mimeType) {
        'image/jpeg' => 'jpg',
        'image/png' => 'png',
        'image/gif' => 'gif',
        'image/webp' => 'webp',
        default => $file->getClientOriginalExtension(),
    };

    $filename = Str::uuid() . '.' . $extension;
    $directory = 'images/' . date('Y/m');
    $path = "{$directory}/{$filename}";

    // บันทึกไฟล์
    Storage::disk('public')->put($path, file_get_contents($file));

    return response()->json([
        'path' => $path,
        'url' => Storage::disk('public')->url($path),
    ]);
}
```

---

## 4. Image Processing

### 4.1 ติดตั้ง Intervention Image

```bash
composer require intervention/image
```

```php
// config/app.php (สำหรับ v2)
'providers' => [
    Intervention\Image\ImageServiceProvider::class,
],
'aliases' => [
    'Image' => Intervention\Image\Facades\Image::class,
],
```

### 4.2 Image Manipulation

```php
use Intervention\Image\Facades\Image;

// Resize
$image = Image::make(Storage::path('uploads/photo.jpg'));
$image->resize(800, 600, function ($constraint) {
    $constraint->aspectRatio();   // รักษาสัดส่วน
    $constraint->upsize();        // ไม่ขยายถ้าเล็กกว่า
});
$image->save(Storage::path('uploads/photo_resized.jpg'), 85);

// Crop (ตัดตรงกลาง)
$image->fit(300, 300);
$image->save(Storage::path('uploads/thumbnail.jpg'));

// Crop custom
$image->crop(200, 200, 100, 50);  // width, height, x, y

// Watermark
$watermark = Image::make(public_path('watermark.png'));
$watermark->opacity(50);
$image->insert($watermark, 'bottom-right', 10, 10);

// Convert format
$image->encode('webp', 85);
$image->save(Storage::path('uploads/photo.webp'));

// Rotate, flip
$image->rotate(90);
$image->flip('h');  // horizontal
$image->flip('v');  // vertical

// Filter
$image->greyscale();
$image->brightness(20);
$image->contrast(30);
$image->blur(15);
```

### 4.3 Image Upload Service

```php
<?php
// app/Services/ImageService.php

namespace App\Services;

use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;
use Intervention\Image\Facades\Image;

class ImageService
{
    private array $sizes = [
        'original' => [null, null],
        'large' => [1200, 900],
        'medium' => [600, 450],
        'thumbnail' => [150, 150],
    ];

    public function upload(
        UploadedFile $file,
        string $directory = 'uploads',
        bool $generateVariants = true
    ): array {
        $filename = Str::uuid();
        $extension = $file->getClientOriginalExtension();
        
        $result = [];

        foreach ($this->sizes as $size => [$width, $height]) {
            if ($size !== 'original' && !$generateVariants) {
                continue;
            }

            $image = Image::make($file);

            if ($width && $height) {
                if ($size === 'thumbnail') {
                    $image->fit($width, $height);
                } else {
                    $image->resize($width, $height, function ($constraint) {
                        $constraint->aspectRatio();
                        $constraint->upsize();
                    });
                }
            }

            $path = "{$directory}/{$size}/{$filename}.{$extension}";
            
            Storage::disk('public')->put(
                $path,
                (string) $image->encode(null, 85)
            );

            $result[$size] = [
                'path' => $path,
                'url' => Storage::disk('public')->url($path),
                'width' => $image->width(),
                'height' => $image->height(),
            ];
        }

        return $result;
    }

    public function delete(array $paths): void
    {
        foreach ($paths as $path) {
            if (Storage::disk('public')->exists($path)) {
                Storage::disk('public')->delete($path);
            }
        }
    }
}
```

---

## 5. File Visibility

```php
// ตั้งค่า visibility ตอน put
Storage::put('file.txt', 'contents', 'public');
Storage::put('private.txt', 'contents', 'private');

// เปลี่ยน visibility ภายหลัง
Storage::setVisibility('file.txt', 'public');
Storage::setVisibility('file.txt', 'private');

// ตรวจสอบ
Storage::visibility('file.txt'); // 'public' หรือ 'private'

// Temporary URL สำหรับไฟล์ private (S3)
$url = Storage::temporaryUrl(
    'private-file.pdf',
    now()->addMinutes(30)
);
```

### 5.1 Signed URL (Local)

```php
// สร้าง signed URL สำหรับ private files
use Illuminate\Support\Facades\URL;

$signedUrl = URL::temporarySignedRoute(
    'files.download',
    now()->addMinutes(30),
    ['path' => 'documents/invoice.pdf']
);
```

```php
// Route
Route::get('/files/download', function (Request $request) {
    if (!$request->hasValidSignature()) {
        abort(401);
    }

    $path = $request->get('path');
    
    // ตรวจสอบว่า path ปลอดภัย
    if (!Storage::disk('local')->exists($path)) {
        abort(404);
    }

    return Storage::disk('local')->download($path);
})->name('files.download');
```

---

## 6. Custom Filesystem Driver

```php
<?php
// app/Providers/FilesystemServiceProvider.php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Facades\Storage;
use League\Flysystem\Filesystem;
use App\Extensions\DropboxAdapter;

class FilesystemServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        Storage::extend('dropbox', function ($app, $config) {
            $client = new \Spatie\Dropbox\Client($config['access_token']);
            $adapter = new \Spatie\FlysystemDropbox\DropboxAdapter($client);
            
            return new Filesystem($adapter, ['case_sensitive' => false]);
        });
    }
}
```

```php
// config/filesystems.php
'disks' => [
    'dropbox' => [
        'driver' => 'dropbox',
        'access_token' => env('DROPBOX_ACCESS_TOKEN'),
    ],
],
```

---

## 7. Workshop: Profile Picture Upload

```php
<?php
// app/Http/Controllers/ProfileController.php

namespace App\Http\Controllers;

use App\Http\Requests\UpdateAvatarRequest;
use App\Services\ImageService;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Storage;

class ProfileController extends Controller
{
    public function __construct(
        private readonly ImageService $imageService
    ) {}

    public function updateAvatar(UpdateAvatarRequest $request): \Illuminate\Http\JsonResponse
    {
        $user = auth()->user();

        // ลบรูปเก่า
        if ($user->avatar_data) {
            $oldPaths = collect(json_decode($user->avatar_data, true))
                ->pluck('path')
                ->toArray();
            $this->imageService->delete($oldPaths);
        }

        // Upload รูปใหม่
        $variants = $this->imageService->upload(
            $request->file('avatar'),
            'avatars',
            true
        );

        // อัพเดต user
        $user->update([
            'avatar' => $variants['thumbnail']['url'],
            'avatar_data' => json_encode($variants),
        ]);

        return response()->json([
            'message' => 'Avatar updated successfully',
            'avatar' => $variants['thumbnail']['url'],
            'avatar_medium' => $variants['medium']['url'],
        ]);
    }

    public function removeAvatar(): \Illuminate\Http\JsonResponse
    {
        $user = auth()->user();

        if ($user->avatar_data) {
            $paths = collect(json_decode($user->avatar_data, true))
                ->pluck('path')
                ->toArray();
            $this->imageService->delete($paths);
        }

        $user->update([
            'avatar' => null,
            'avatar_data' => null,
        ]);

        return response()->json(['message' => 'Avatar removed']);
    }
}
```

```php
<?php
// app/Http/Requests/UpdateAvatarRequest.php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class UpdateAvatarRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'avatar' => [
                'required',
                'image',
                'mimes:jpeg,jpg,png,webp',
                'max:2048',  // 2MB
                'dimensions:min_width=100,min_height=100,max_width=2000,max_height=2000',
            ],
        ];
    }

    public function messages(): array
    {
        return [
            'avatar.required' => 'กรุณาเลือกรูปภาพ',
            'avatar.image' => 'ไฟล์ต้องเป็นรูปภาพ',
            'avatar.mimes' => 'รองรับเฉพาะ JPEG, PNG, WebP',
            'avatar.max' => 'ขนาดไฟล์ต้องไม่เกิน 2MB',
            'avatar.dimensions' => 'ขนาดรูปต้องอยู่ระหว่าง 100x100 ถึง 2000x2000 px',
        ];
    }
}
```

---

## 8. Workshop: Document Management System

```php
<?php
// app/Models/Document.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\Storage;

class Document extends Model
{
    protected $fillable = [
        'user_id', 'name', 'path', 'disk',
        'size', 'mime_type', 'is_public',
        'metadata', 'expires_at',
    ];

    protected $casts = [
        'metadata' => 'array',
        'is_public' => 'boolean',
        'expires_at' => 'datetime',
    ];

    public function getUrlAttribute(): string
    {
        if ($this->is_public) {
            return Storage::disk($this->disk)->url($this->path);
        }

        // Private file - signed URL
        return \URL::temporarySignedRoute(
            'documents.download',
            now()->addHours(1),
            ['document' => $this->id]
        );
    }

    public function isExpired(): bool
    {
        return $this->expires_at && $this->expires_at->isPast();
    }

    protected static function booted(): void
    {
        // ลบไฟล์เมื่อ delete record
        static::deleting(function (Document $document) {
            if (Storage::disk($document->disk)->exists($document->path)) {
                Storage::disk($document->disk)->delete($document->path);
            }
        });
    }
}
```

```php
<?php
// app/Http/Controllers/DocumentController.php

namespace App\Http\Controllers;

use App\Models\Document;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;

class DocumentController extends Controller
{
    public function index(Request $request): \Illuminate\Http\JsonResponse
    {
        $documents = Document::where('user_id', auth()->id())
            ->when($request->search, fn($q) => $q->where('name', 'like', "%{$request->search}%"))
            ->when($request->type, fn($q) => $q->where('mime_type', 'like', "{$request->type}%"))
            ->latest()
            ->paginate(20);

        return response()->json($documents);
    }

    public function store(Request $request): \Illuminate\Http\JsonResponse
    {
        $request->validate([
            'document' => 'required|file|max:20480', // 20MB
            'name' => 'nullable|string|max:255',
            'is_public' => 'boolean',
            'expires_in_days' => 'nullable|integer|min:1|max:365',
        ]);

        $file = $request->file('document');
        $disk = $request->boolean('is_public') ? 'public' : 'local';
        $directory = 'documents/' . auth()->id() . '/' . date('Y/m');
        $filename = Str::uuid() . '.' . $file->getClientOriginalExtension();
        $path = "{$directory}/{$filename}";

        Storage::disk($disk)->put($path, file_get_contents($file));

        $document = Document::create([
            'user_id' => auth()->id(),
            'name' => $request->name ?: $file->getClientOriginalName(),
            'path' => $path,
            'disk' => $disk,
            'size' => $file->getSize(),
            'mime_type' => $file->getMimeType(),
            'is_public' => $request->boolean('is_public'),
            'expires_at' => $request->expires_in_days 
                ? now()->addDays($request->expires_in_days) 
                : null,
        ]);

        return response()->json($document, 201);
    }

    public function download(Request $request, Document $document): \Illuminate\Http\Response
    {
        // ตรวจสอบว่าเป็นเจ้าของหรือ public
        if (!$document->is_public && $document->user_id !== auth()->id()) {
            abort(403);
        }

        if ($document->isExpired()) {
            abort(410, 'Document has expired');
        }

        // ตรวจสอบ signed URL (สำหรับ private)
        if (!$document->is_public && !$request->hasValidSignature()) {
            abort(401);
        }

        if (!Storage::disk($document->disk)->exists($document->path)) {
            abort(404, 'File not found');
        }

        return Storage::disk($document->disk)->download($document->path, $document->name);
    }

    public function destroy(Document $document): \Illuminate\Http\JsonResponse
    {
        $this->authorize('delete', $document);
        
        $document->delete(); // Observer จะลบไฟล์ให้

        return response()->json(['message' => 'Document deleted']);
    }
}
```

---

## Quiz

### คำถาม 1
ความแตกต่างระหว่าง `local` disk และ `public` disk คืออะไร?

**A)** local เร็วกว่า public  
**B)** local เข้าถึงไม่ได้จาก browser, public เข้าถึงได้ผ่าน URL  
**C)** ไม่ต่างกัน  
**D)** public ใช้ S3 เสมอ  

**เฉลย: B** - `local` เก็บใน `storage/app` ที่ไม่มี public URL, `public` เก็บใน `storage/app/public` ที่ symbolic link ไปยัง `public/storage`

---

### คำถาม 2
`Storage::temporaryUrl()` ใช้เพื่ออะไร?

**A)** Upload ไฟล์ชั่วคราว  
**B)** สร้าง URL ที่หมดอายุสำหรับไฟล์ private บน S3  
**C)** สร้าง thumbnail  
**D)** Cache URL  

**เฉลย: B** - `temporaryUrl()` สร้าง pre-signed URL สำหรับเข้าถึงไฟล์ private บน S3 โดยกำหนดวันหมดอายุได้

---

### คำถาม 3
`php artisan storage:link` ทำอะไร?

**A)** ลิงก์ S3 bucket  
**B)** สร้าง symbolic link จาก `public/storage` ไปยัง `storage/app/public`  
**C)** sync ไฟล์ระหว่าง disks  
**D)** เปิด storage permission  

**เฉลย: B** - สร้าง symbolic link เพื่อให้ไฟล์ใน `storage/app/public` สามารถเข้าถึงได้ผ่าน browser

---

### คำถาม 4
`$constraint->upsize()` ใน Intervention Image ทำอะไร?

**A)** ขยายรูปให้ใหญ่ขึ้น  
**B)** ป้องกันไม่ให้รูปถูกขยายถ้ารูปเล็กกว่า target size  
**C)** เพิ่มความละเอียด  
**D)** แปลง format  

**เฉลย: B** - `upsize()` ป้องกัน pixelation โดยไม่ขยายรูปถ้ารูปมีขนาดเล็กกว่า target

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- ✅ ตั้งค่า Storage disks (Local, Public, S3, MinIO)
- ✅ การ Upload ไฟล์และ Image validation
- ✅ Image processing ด้วย Intervention Image
- ✅ File Visibility และ Signed URLs
- ✅ Custom Filesystem Driver
- ✅ Workshop: Profile Picture Upload และ Document Management System

---

## ลิงก์ไป Part ถัดไป

➡️ **[Part 041: Laravel API Development](part-041-laravel-api.md)**  
เรียนรู้เรื่อง RESTful API Design, API Resources, Authentication ด้วย Sanctum, Rate Limiting
