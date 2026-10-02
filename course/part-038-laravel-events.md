# Part 038: Laravel Events & Broadcasting

**ระดับ:** สูง / มืออาชีพ  
**เวลาเรียน:** 4-5 ชั่วโมง  
**ความต้องการก่อนเรียน:** Part 037, WebSocket พื้นฐาน

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ Events & Listeners ในรูปแบบต่างๆ
- ตั้งค่า Event Discovery อัตโนมัติ
- สร้าง Queued Listeners สำหรับ background processing
- สร้าง Event Subscribers
- ตั้งค่า Real-time Broadcasting ด้วย Pusher/Soketi
- Workshop: Activity log และ Real-time notifications

---

## 1. Events & Listeners คืออะไร?

Events เป็น Observer Pattern ที่ช่วยให้ code แยกส่วนกัน (decoupled) เมื่อเกิดบางสิ่งในระบบ เราสามารถ "fire" event แล้วให้ listeners ต่างๆ จัดการตามต้องการ

```
Action (Register User)
    → Fire Event: UserRegistered
        → Listener 1: SendWelcomeEmail
        → Listener 2: CreateUserProfile
        → Listener 3: LogActivity
        → Listener 4: SendSlackNotification
```

ข้อดี:
- Code แยกส่วนกัน เพิ่ม/ลด listener ได้ง่าย
- ทดสอบง่ายกว่า
- ลด dependency ระหว่าง modules

---

## 2. สร้าง Event

```bash
php artisan make:event UserRegistered
```

```php
<?php
// app/Events/UserRegistered.php

namespace App\Events;

use App\Models\User;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class UserRegistered
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(
        public readonly User $user,
        public readonly string $registrationSource = 'web', // web, api, social
        public readonly array $metadata = []
    ) {}
}
```

---

## 3. สร้าง Listeners

```bash
php artisan make:listener SendWelcomeEmail --event=UserRegistered
```

```php
<?php
// app/Listeners/SendWelcomeEmail.php

namespace App\Listeners;

use App\Events\UserRegistered;
use App\Mail\WelcomeEmail;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Support\Facades\Mail;
use Throwable;

class SendWelcomeEmail implements ShouldQueue
{
    use InteractsWithQueue;

    // Queue settings
    public string $queue = 'emails';
    public int $delay = 0;
    public int $tries = 3;

    /**
     * Handle the event.
     */
    public function handle(UserRegistered $event): void
    {
        Mail::to($event->user->email)
            ->send(new WelcomeEmail($event->user));
    }

    /**
     * Handle failure.
     */
    public function failed(UserRegistered $event, Throwable $exception): void
    {
        \Log::error('SendWelcomeEmail listener failed', [
            'user_id' => $event->user->id,
            'error' => $exception->getMessage(),
        ]);
    }
}
```

```php
<?php
// app/Listeners/CreateUserProfile.php

namespace App\Listeners;

use App\Events\UserRegistered;
use App\Models\UserProfile;

class CreateUserProfile
{
    public function handle(UserRegistered $event): void
    {
        // สร้าง profile เมื่อ user ลงทะเบียน
        UserProfile::create([
            'user_id' => $event->user->id,
            'avatar' => 'default.jpg',
            'bio' => '',
            'timezone' => 'Asia/Bangkok',
        ]);
    }
}
```

---

## 4. Register Events & Listeners

### 4.1 Manual Registration (Laravel 10 และก่อนหน้า)

```php
<?php
// app/Providers/EventServiceProvider.php

namespace App\Providers;

use App\Events\UserRegistered;
use App\Events\OrderPlaced;
use App\Events\OrderShipped;
use App\Listeners\SendWelcomeEmail;
use App\Listeners\CreateUserProfile;
use App\Listeners\LogActivity;
use App\Listeners\SendOrderConfirmation;
use App\Listeners\UpdateInventory;
use App\Listeners\NotifyShipping;
use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;

class EventServiceProvider extends ServiceProvider
{
    protected $listen = [
        // User Events
        UserRegistered::class => [
            SendWelcomeEmail::class,
            CreateUserProfile::class,
            LogActivity::class,
        ],
        
        // Order Events
        OrderPlaced::class => [
            SendOrderConfirmation::class,
            UpdateInventory::class,
        ],
        
        OrderShipped::class => [
            NotifyShipping::class,
        ],
    ];
}
```

### 4.2 Event Discovery (อัตโนมัติ)

```php
// app/Providers/EventServiceProvider.php

class EventServiceProvider extends ServiceProvider
{
    /**
     * Determine if events and listeners should be automatically discovered.
     */
    public function shouldDiscoverEvents(): bool
    {
        return true; // เปิด discovery
    }

    /**
     * Get the listener directories that should be used to discover events.
     */
    protected function discoverEventsWithin(): array
    {
        return [
            $this->app->path('Listeners'), // ดูจาก app/Listeners/
        ];
    }
}
```

เมื่อเปิด Discovery Laravel จะอ่าน type hint ของ `handle()` method ใน Listener อัตโนมัติ:

```php
// ไม่ต้อง register ใน $listen
// Laravel รู้จาก type hint: UserRegistered $event

class SendWelcomeEmail
{
    public function handle(UserRegistered $event): void  // <-- type hint
    {
        // ...
    }
}
```

```bash
# Cache discovered events
php artisan event:cache
php artisan event:clear
```

---

## 5. Fire Events

```php
// วิธีที่ 1: ใช้ event() helper
event(new UserRegistered($user));

// วิธีที่ 2: ใช้ Event facade
use Illuminate\Support\Facades\Event;
Event::dispatch(new UserRegistered($user));

// วิธีที่ 3: ใช้ static dispatch บน Event class
UserRegistered::dispatch($user);

// ตัวอย่างใน Controller
class AuthController extends Controller
{
    public function register(RegisterRequest $request): JsonResponse
    {
        $user = User::create($request->validated());

        // Fire event - listeners จะจัดการส่วนที่เหลือ
        UserRegistered::dispatch($user, 'web');

        return response()->json([
            'user' => new UserResource($user),
            'message' => 'Registration successful',
        ], 201);
    }
}
```

---

## 6. Event Subscribers

Subscriber รวม logic สำหรับหลาย events ไว้ในไฟล์เดียว:

```php
<?php
// app/Listeners/UserEventSubscriber.php

namespace App\Listeners;

use App\Events\UserRegistered;
use App\Events\UserLoggedIn;
use App\Events\UserLoggedOut;
use App\Events\UserPasswordChanged;
use Illuminate\Events\Dispatcher;

class UserEventSubscriber
{
    public function handleUserRegistered(UserRegistered $event): void
    {
        \Log::info('User registered', ['user_id' => $event->user->id]);
        
        // บันทึก activity
        activity()
            ->causedBy($event->user)
            ->log('User registered');
    }

    public function handleUserLoggedIn(UserLoggedIn $event): void
    {
        // อัพเดต last login
        $event->user->update([
            'last_login_at' => now(),
            'last_login_ip' => request()->ip(),
        ]);
        
        \Log::info('User logged in', [
            'user_id' => $event->user->id,
            'ip' => request()->ip(),
        ]);
    }

    public function handleUserLoggedOut(UserLoggedOut $event): void
    {
        \Log::info('User logged out', ['user_id' => $event->user->id]);
    }

    public function handleUserPasswordChanged(UserPasswordChanged $event): void
    {
        // ส่ง security alert
        $event->user->notify(new \App\Notifications\PasswordChanged());
        
        \Log::warning('User password changed', ['user_id' => $event->user->id]);
    }

    /**
     * Register the listeners for the subscriber.
     */
    public function subscribe(Dispatcher $events): array
    {
        return [
            UserRegistered::class => 'handleUserRegistered',
            UserLoggedIn::class => 'handleUserLoggedIn',
            UserLoggedOut::class => 'handleUserLoggedOut',
            UserPasswordChanged::class => 'handleUserPasswordChanged',
        ];
    }
}
```

```php
// app/Providers/EventServiceProvider.php

protected $subscribe = [
    UserEventSubscriber::class,
    OrderEventSubscriber::class,
];
```

---

## 7. Broadcasting (Real-time)

Broadcasting ส่ง event ไปยัง client ผ่าน WebSocket แบบ real-time

### 7.1 ติดตั้ง Soketi (Alternative ของ Pusher ที่ run locally)

```bash
# ติดตั้ง Node.js package
npm install -g @soketi/soketi

# รัน Soketi
soketi start --config=/path/to/config.json
```

```json
// soketi-config.json
{
    "debug": false,
    "port": 6001,
    "appManager.driver": "array",
    "appManager.array.apps": [
        {
            "id": "my-app",
            "key": "my-app-key",
            "secret": "my-app-secret",
            "enableClientMessages": true,
            "maxConnections": 100
        }
    ]
}
```

### 7.2 ตั้งค่า Laravel Broadcasting

```env
BROADCAST_DRIVER=pusher

PUSHER_APP_ID=my-app
PUSHER_APP_KEY=my-app-key
PUSHER_APP_SECRET=my-app-secret
PUSHER_HOST=127.0.0.1
PUSHER_PORT=6001
PUSHER_SCHEME=http
PUSHER_APP_CLUSTER=mt1
```

```bash
composer require pusher/pusher-php-server
```

```php
// config/broadcasting.php
'pusher' => [
    'driver' => 'pusher',
    'key' => env('PUSHER_APP_KEY'),
    'secret' => env('PUSHER_APP_SECRET'),
    'app_id' => env('PUSHER_APP_ID'),
    'options' => [
        'cluster' => env('PUSHER_APP_CLUSTER'),
        'host' => env('PUSHER_HOST', '127.0.0.1'),
        'port' => env('PUSHER_PORT', 443),
        'scheme' => env('PUSHER_SCHEME', 'https'),
        'useTLS' => env('PUSHER_SCHEME', 'https') === 'https',
    ],
],
```

### 7.3 Broadcast Event

```php
<?php
// app/Events/OrderStatusUpdated.php

namespace App\Events;

use App\Models\Order;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class OrderStatusUpdated implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(
        public readonly Order $order
    ) {}

    /**
     * ระบุ channel ที่จะ broadcast ไป
     */
    public function broadcastOn(): array
    {
        return [
            // Public channel - ทุกคนดูได้
            new Channel('orders'),
            
            // Private channel - เฉพาะ user ที่ authorized
            new PrivateChannel("orders.{$this->order->user_id}"),
        ];
    }

    /**
     * ชื่อของ event ที่จะส่ง (default: class name)
     */
    public function broadcastAs(): string
    {
        return 'order.status.updated';
    }

    /**
     * ข้อมูลที่จะส่ง
     */
    public function broadcastWith(): array
    {
        return [
            'order_id' => $this->order->id,
            'status' => $this->order->status,
            'message' => $this->order->getStatusMessage(),
            'updated_at' => $this->order->updated_at->toISOString(),
        ];
    }

    /**
     * เงื่อนไขก่อน broadcast
     */
    public function broadcastWhen(): bool
    {
        return $this->order->status !== 'processing'; // ไม่ broadcast ตอน processing
    }
}
```

### 7.4 Channel Authorization

```php
// routes/channels.php

use App\Models\Order;

// Private Channel: ตรวจสอบว่า user เป็นเจ้าของ order
Broadcast::channel('orders.{userId}', function ($user, $userId) {
    return (int) $user->id === (int) $userId;
});

// Presence Channel: ส่ง user info ไปด้วย
Broadcast::channel('chat.{roomId}', function ($user, $roomId) {
    if ($user->canJoinRoom($roomId)) {
        return [
            'id' => $user->id,
            'name' => $user->name,
            'avatar' => $user->avatar_url,
        ];
    }
    return false;
});
```

### 7.5 Frontend: Laravel Echo

```bash
npm install --save-dev laravel-echo pusher-js
```

```javascript
// resources/js/bootstrap.js

import Echo from 'laravel-echo';
import Pusher from 'pusher-js';

window.Pusher = Pusher;

window.Echo = new Echo({
    broadcaster: 'pusher',
    key: import.meta.env.VITE_PUSHER_APP_KEY,
    cluster: import.meta.env.VITE_PUSHER_APP_CLUSTER ?? 'mt1',
    wsHost: import.meta.env.VITE_PUSHER_HOST ?? `ws-${import.meta.env.VITE_PUSHER_APP_CLUSTER}.pusher.com`,
    wsPort: import.meta.env.VITE_PUSHER_PORT ?? 80,
    wssPort: import.meta.env.VITE_PUSHER_PORT ?? 443,
    forceTLS: (import.meta.env.VITE_PUSHER_SCHEME ?? 'https') === 'https',
    enabledTransports: ['ws', 'wss'],
});
```

```javascript
// ฟัง Public Channel
Echo.channel('orders')
    .listen('.order.status.updated', (event) => {
        console.log('Order updated:', event);
        updateOrderUI(event);
    });

// ฟัง Private Channel
Echo.private(`orders.${userId}`)
    .listen('.order.status.updated', (event) => {
        showNotification(`Order ${event.order_id} is now ${event.status}`);
    });

// Presence Channel (ใครออนไลน์อยู่)
Echo.join(`chat.${roomId}`)
    .here((users) => {
        console.log('Users in room:', users);
    })
    .joining((user) => {
        console.log(`${user.name} joined`);
    })
    .leaving((user) => {
        console.log(`${user.name} left`);
    })
    .listen('.message.sent', (event) => {
        addMessage(event);
    });
```

---

## 8. Workshop: Activity Log

### 8.1 สร้าง Event

```php
<?php
// app/Events/ActivityLogged.php

namespace App\Events;

use App\Models\User;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class ActivityLogged
{
    use Dispatchable, SerializesModels;

    public function __construct(
        public readonly ?User $user,
        public readonly string $action,
        public readonly string $description,
        public readonly ?string $subject_type = null,
        public readonly ?int $subject_id = null,
        public readonly array $properties = []
    ) {}
}
```

### 8.2 Activity Logger Service

```php
<?php
// app/Services/ActivityLogger.php

namespace App\Services;

use App\Events\ActivityLogged;
use App\Models\ActivityLog;
use App\Models\User;
use Illuminate\Database\Eloquent\Model;

class ActivityLogger
{
    private ?User $user = null;
    private string $action = '';
    private string $description = '';
    private ?Model $subject = null;
    private array $properties = [];

    public function causedBy(?User $user): static
    {
        $this->user = $user;
        return $this;
    }

    public function performedOn(Model $subject): static
    {
        $this->subject = $subject;
        return $this;
    }

    public function withProperties(array $properties): static
    {
        $this->properties = $properties;
        return $this;
    }

    public function log(string $action, string $description = ''): ActivityLog
    {
        $log = ActivityLog::create([
            'user_id' => $this->user?->id,
            'action' => $action,
            'description' => $description ?: $action,
            'subject_type' => $this->subject ? get_class($this->subject) : null,
            'subject_id' => $this->subject?->id,
            'properties' => $this->properties,
            'ip_address' => request()->ip(),
            'user_agent' => request()->userAgent(),
        ]);

        // Fire event
        ActivityLogged::dispatch(
            $this->user,
            $action,
            $description,
            $this->subject ? get_class($this->subject) : null,
            $this->subject?->id,
            $this->properties
        );

        return $log;
    }
}
```

### 8.3 ตัวอย่างการใช้งาน

```php
// ใน Controller หรือ Observer

// Register Service in AppServiceProvider
app(\App\Services\ActivityLogger::class)
    ->causedBy(auth()->user())
    ->performedOn($post)
    ->withProperties(['old_title' => $oldTitle, 'new_title' => $post->title])
    ->log('post.updated', "Updated post: {$post->title}");
```

---

## 9. Workshop: Real-time Notifications

```php
<?php
// app/Events/NotificationCreated.php

namespace App\Events;

use App\Models\Notification;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class NotificationCreated implements ShouldBroadcast
{
    use Dispatchable, SerializesModels;

    public function __construct(
        public readonly Notification $notification
    ) {}

    public function broadcastOn(): array
    {
        return [
            new PrivateChannel("user.{$this->notification->user_id}.notifications"),
        ];
    }

    public function broadcastAs(): string
    {
        return 'notification.created';
    }

    public function broadcastWith(): array
    {
        return [
            'id' => $this->notification->id,
            'type' => $this->notification->type,
            'title' => $this->notification->title,
            'message' => $this->notification->message,
            'data' => $this->notification->data,
            'created_at' => $this->notification->created_at->toISOString(),
        ];
    }
}
```

```php
<?php
// app/Services/NotificationService.php

namespace App\Services;

use App\Events\NotificationCreated;
use App\Models\Notification;
use App\Models\User;

class NotificationService
{
    public function send(
        User $user,
        string $type,
        string $title,
        string $message,
        array $data = []
    ): Notification {
        $notification = Notification::create([
            'user_id' => $user->id,
            'type' => $type,
            'title' => $title,
            'message' => $message,
            'data' => $data,
            'read_at' => null,
        ]);

        // Broadcast real-time
        NotificationCreated::dispatch($notification);

        return $notification;
    }

    public function markAsRead(int $notificationId, User $user): void
    {
        Notification::where('id', $notificationId)
            ->where('user_id', $user->id)
            ->update(['read_at' => now()]);
    }

    public function getUnread(User $user): \Illuminate\Database\Eloquent\Collection
    {
        return Notification::where('user_id', $user->id)
            ->whereNull('read_at')
            ->latest()
            ->get();
    }
}
```

```javascript
// Frontend: ฟัง notifications

const userId = document.querySelector('meta[name="user-id"]').content;

Echo.private(`user.${userId}.notifications`)
    .listen('.notification.created', (notification) => {
        // แสดง notification badge
        updateNotificationCount();
        
        // แสดง toast
        showToast(notification.title, notification.message);
        
        // เพิ่มใน dropdown
        prependToNotificationList(notification);
    });

function updateNotificationCount() {
    const badge = document.querySelector('.notification-badge');
    const current = parseInt(badge.textContent || '0');
    badge.textContent = current + 1;
    badge.style.display = 'block';
}
```

---

## Quiz

### คำถาม 1
Event Discovery ใน Laravel ทำงานอย่างไร?

**A)** อ่านจาก config file  
**B)** อ่าน type hint ของ `handle()` method ใน Listener อัตโนมัติ  
**C)** สแกน route files  
**D)** ใช้ annotation  

**เฉลย: B** - Laravel อ่าน type hint ของ parameter ใน `handle()` method เพื่อรู้ว่า Listener นี้ subscribe กับ Event ใด

---

### คำถาม 2
`ShouldBroadcast` interface ทำให้ Event ทำอะไรได้พิเศษ?

**A)** ส่ง email อัตโนมัติ  
**B)** บันทึก event ลง database  
**C)** ส่ง event ผ่าน WebSocket ไปยัง frontend แบบ real-time  
**D)** รัน event ใน background  

**เฉลย: C** - `ShouldBroadcast` ทำให้ event ถูกส่งผ่าน WebSocket (Pusher/Soketi) ไปยัง frontend

---

### คำถาม 3
PrivateChannel vs PublicChannel ต่างกันอย่างไร?

**A)** PrivateChannel เร็วกว่า  
**B)** PrivateChannel ต้องการ authentication ก่อน subscribe  
**C)** PublicChannel เข้ารหัสข้อมูล  
**D)** ไม่ต่างกัน  

**เฉลย: B** - PrivateChannel ต้องผ่าน authorization ใน `routes/channels.php` ก่อนจึงจะ subscribe ได้

---

### คำถาม 4
Event Subscriber ต่างจาก Event Listener ปกติอย่างไร?

**A)** Subscriber เร็วกว่า  
**B)** Subscriber รวม handlers สำหรับหลาย events ไว้ในไฟล์เดียว  
**C)** Subscriber ไม่รองรับ queued execution  
**D)** Subscriber ใช้ได้เฉพาะกับ broadcast events  

**เฉลย: B** - Event Subscriber รวม logic สำหรับหลาย events ในคลาสเดียว ผ่าน `subscribe()` method

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- ✅ Events & Listeners พื้นฐานและ advanced
- ✅ Event Discovery อัตโนมัติ
- ✅ Queued Listeners สำหรับ background processing
- ✅ Event Subscribers สำหรับจัดกลุ่ม
- ✅ Real-time Broadcasting ด้วย Soketi/Pusher
- ✅ Workshop: Activity Log System และ Real-time Notifications

---

## ลิงก์ไป Part ถัดไป

➡️ **[Part 039: Laravel Mail & Notifications](part-039-laravel-mail.md)**  
เรียนรู้เรื่อง Mailable classes, Markdown emails, Mail queuing และ Notifications ผ่านช่องทางต่างๆ
