# Part 044: Laravel Broadcasting — Real-time Communication

**ระดับ: มืออาชีพ (Professional)**
**เวลาเรียน: 5-6 ชั่วโมง**

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ตั้งค่า Laravel Echo กับ JavaScript Frontend
- ใช้งาน Private และ Presence Channels
- Broadcasting Events ไปยัง Clients แบบ Real-time
- ตั้งค่า WebSocket Server ด้วย Soketi
- สร้าง Real-time Chat Application ด้วย Laravel + Echo

---

## 1. Broadcasting คืออะไร?

Laravel Broadcasting ช่วยให้ Server สามารถส่งข้อมูลไปยัง Client แบบ Real-time ผ่าน WebSocket Protocol แทนที่จะต้องให้ Client Polling ตลอดเวลา

**Use Cases:**
- Real-time Chat
- Live Notifications
- Online User Presence
- Live Updates (Stock prices, Sports scores)
- Collaborative Editing

```
Client (Browser)          Laravel App           WebSocket Server (Soketi/Pusher)
      |                        |                         |
      |-- Subscribe Channel -->|                         |
      |                        |-- Auth Request -------->|
      |<-- WebSocket Connect --|                         |
      |                        |                         |
      |                        |-- Fire Event ---------> |
      |<-- Push Event ---------|                         |
      |                        |                         |
```

---

## 2. ติดตั้งและตั้งค่า

### 2.1 ติดตั้ง Dependencies

```bash
# Laravel side
composer require pusher/pusher-php-server

# JavaScript side
npm install --save-dev laravel-echo pusher-js
```

### 2.2 ตั้งค่า .env

```bash
BROADCAST_DRIVER=pusher

PUSHER_APP_ID=local
PUSHER_APP_KEY=local
PUSHER_APP_SECRET=local
PUSHER_HOST=127.0.0.1
PUSHER_PORT=6001
PUSHER_SCHEME=http
PUSHER_APP_CLUSTER=mt1

VITE_PUSHER_APP_KEY="${PUSHER_APP_KEY}"
VITE_PUSHER_HOST="${PUSHER_HOST}"
VITE_PUSHER_PORT="${PUSHER_PORT}"
VITE_PUSHER_SCHEME="${PUSHER_SCHEME}"
VITE_PUSHER_APP_CLUSTER="${PUSHER_APP_CLUSTER}"
```

### 2.3 ตั้งค่า config/broadcasting.php

```php
// config/broadcasting.php
return [
    'default' => env('BROADCAST_DRIVER', 'null'),
    
    'connections' => [
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
                'encrypted' => true,
                'useTLS' => env('PUSHER_SCHEME', 'https') === 'https',
            ],
            'client_options' => [
                // Guzzle client options: https://docs.guzzlephp.org/en/stable/request-options.html
            ],
        ],
        
        'ably' => [
            'driver' => 'ably',
            'key' => env('ABLY_KEY'),
        ],
        
        'log' => [
            'driver' => 'log',
        ],
        
        'null' => [
            'driver' => 'null',
        ],
    ],
];
```

### 2.4 ตั้งค่า Laravel Echo ใน JavaScript

```javascript
// resources/js/bootstrap.js
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';

window.Pusher = Pusher;

window.Echo = new Echo({
    broadcaster: 'pusher',
    key: import.meta.env.VITE_PUSHER_APP_KEY,
    cluster: import.meta.env.VITE_PUSHER_APP_CLUSTER,
    wsHost: import.meta.env.VITE_PUSHER_HOST,
    wsPort: import.meta.env.VITE_PUSHER_PORT ?? 80,
    wssPort: import.meta.env.VITE_PUSHER_PORT ?? 443,
    forceTLS: (import.meta.env.VITE_PUSHER_SCHEME ?? 'https') === 'https',
    enabledTransports: ['ws', 'wss'],
    authEndpoint: '/broadcasting/auth',
    auth: {
        headers: {
            'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]')?.content,
        },
    },
});
```

---

## 3. Soketi — Self-hosted WebSocket Server

Soketi เป็น Open-source, Pusher-compatible WebSocket Server ที่ใช้งานฟรี

### 3.1 ติดตั้ง Soketi

```bash
# ติดตั้งด้วย npm
npm install -g @soketi/soketi

# หรือด้วย Docker
docker pull quay.io/soketi/soketi:latest
```

### 3.2 ตั้งค่า Soketi

```json
// soketi.json
{
    "debug": true,
    "port": 6001,
    "appManager.driver": "array",
    "appManager.array.apps": [
        {
            "id": "local",
            "key": "local",
            "secret": "local",
            "webhooks": [],
            "maxConnections": -1,
            "enableClientMessages": false,
            "enabled": true,
            "maxBackendEventsPerSecond": -1,
            "maxClientEventsPerSecond": -1,
            "maxReadRequestsPerSecond": -1,
            "hasRestrictedChannels": false
        }
    ]
}
```

### 3.3 รัน Soketi

```bash
# รันโดยตรง
soketi start --config=soketi.json

# รันด้วย Docker
docker run -p 6001:6001 -p 9601:9601 \
    -e SOKETI_DEBUG=1 \
    -e SOKETI_DEFAULT_APP_ID=local \
    -e SOKETI_DEFAULT_APP_KEY=local \
    -e SOKETI_DEFAULT_APP_SECRET=local \
    quay.io/soketi/soketi:latest
```

### 3.4 Soketi ด้วย Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8000:8000"
    depends_on:
      - redis
      - soketi
    environment:
      BROADCAST_DRIVER: pusher
      PUSHER_APP_ID: local
      PUSHER_APP_KEY: local
      PUSHER_APP_SECRET: local
      PUSHER_HOST: soketi
      PUSHER_PORT: 6001
      PUSHER_SCHEME: http
    
  soketi:
    image: quay.io/soketi/soketi:latest
    ports:
      - "6001:6001"
      - "9601:9601"
    environment:
      SOKETI_DEBUG: 1
      SOKETI_DEFAULT_APP_ID: local
      SOKETI_DEFAULT_APP_KEY: local
      SOKETI_DEFAULT_APP_SECRET: local
      SOKETI_DB_REDIS_HOST: redis
      SOKETI_DB_REDIS_PORT: 6379
    depends_on:
      - redis
  
  redis:
    image: redis:alpine
    ports:
      - "6379:6379"
```

---

## 4. Channel Types

### 4.1 Public Channels

Channel ที่ทุกคนสามารถ Subscribe ได้โดยไม่ต้องตรวจสอบ

```php
// app/Events/NewPost.php
namespace App\Events;

use App\Models\Post;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class NewPost implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;
    
    public function __construct(
        public readonly Post $post
    ) {}
    
    public function broadcastOn(): array
    {
        return [
            new Channel('posts'),
        ];
    }
    
    public function broadcastAs(): string
    {
        return 'new-post';
    }
    
    public function broadcastWith(): array
    {
        return [
            'id' => $this->post->id,
            'title' => $this->post->title,
            'author' => $this->post->author->name,
            'created_at' => $this->post->created_at->toISOString(),
        ];
    }
}
```

**JavaScript - Subscribe Public Channel:**
```javascript
// Subscribe to public channel
Echo.channel('posts')
    .listen('.new-post', (event) => {
        console.log('New post:', event);
        addPostToList(event);
    });
```

### 4.2 Private Channels

Channel ที่ต้องตรวจสอบ Authorization ก่อน Subscribe

```php
// routes/channels.php
use App\Models\User;

// Authorize Private Channel
Broadcast::channel('user.{userId}', function (User $user, int $userId) {
    return (int) $user->id === (int) $userId;
});

// Authorize Channel ด้วย Logic ซับซ้อน
Broadcast::channel('chat.{chatId}', function (User $user, int $chatId) {
    $chat = Chat::find($chatId);
    return $chat && $chat->members->contains($user->id);
});
```

```php
// app/Events/MessageSent.php
use Illuminate\Broadcasting\PrivateChannel;

class MessageSent implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;
    
    public function __construct(
        public readonly Message $message
    ) {}
    
    public function broadcastOn(): array
    {
        return [
            new PrivateChannel('chat.' . $this->message->chat_id),
        ];
    }
    
    public function broadcastAs(): string
    {
        return 'message.sent';
    }
    
    public function broadcastWith(): array
    {
        return [
            'id' => $this->message->id,
            'content' => $this->message->content,
            'user' => [
                'id' => $this->message->user->id,
                'name' => $this->message->user->name,
                'avatar' => $this->message->user->avatar_url,
            ],
            'created_at' => $this->message->created_at->toISOString(),
        ];
    }
}
```

**JavaScript - Subscribe Private Channel:**
```javascript
// Subscribe ต้องมี Auth Token
Echo.private(`chat.${chatId}`)
    .listen('.message.sent', (event) => {
        appendMessage(event);
    })
    .error((error) => {
        console.error('Channel authorization failed:', error);
    });
```

### 4.3 Presence Channels

Channel พิเศษที่ติดตาม Online Users ในช่อง

```php
// routes/channels.php
Broadcast::channel('room.{roomId}', function (User $user, int $roomId) {
    $room = Room::find($roomId);
    
    if ($room && $room->members->contains($user->id)) {
        return [
            'id' => $user->id,
            'name' => $user->name,
            'avatar' => $user->avatar_url,
            'status' => 'online',
        ];
    }
    
    return false;
});
```

```php
// app/Events/UserJoinedRoom.php
use Illuminate\Broadcasting\PresenceChannel;

class UserJoinedRoom implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;
    
    public function __construct(
        public readonly User $user,
        public readonly int $roomId
    ) {}
    
    public function broadcastOn(): array
    {
        return [
            new PresenceChannel('room.' . $this->roomId),
        ];
    }
}
```

**JavaScript - Subscribe Presence Channel:**
```javascript
Echo.join(`room.${roomId}`)
    .here((users) => {
        // รายการ users ที่ online ตอนนี้
        console.log('Current online users:', users);
        setOnlineUsers(users);
    })
    .joining((user) => {
        // มี user ใหม่เข้ามา
        console.log('User joined:', user);
        addOnlineUser(user);
    })
    .leaving((user) => {
        // user ออกไปแล้ว
        console.log('User left:', user);
        removeOnlineUser(user);
    })
    .listen('.message.sent', (event) => {
        appendMessage(event);
    })
    .error((error) => {
        console.error('Presence channel error:', error);
    });
```

---

## 5. Broadcasting Events

### 5.1 ShouldBroadcast vs ShouldBroadcastNow

```php
// ShouldBroadcast - ส่งผ่าน Queue (ปกติ)
class OrderStatusUpdated implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;
    
    public function __construct(
        public readonly Order $order
    ) {}
    
    public function broadcastOn(): array
    {
        return [
            new PrivateChannel('user.' . $this->order->user_id),
            new PrivateChannel('order.' . $this->order->id),
        ];
    }
    
    // ส่งเฉพาะ fields ที่จำเป็น
    public function broadcastWith(): array
    {
        return [
            'order_id' => $this->order->id,
            'status' => $this->order->status,
            'status_label' => $this->order->status_label,
            'updated_at' => $this->order->updated_at->toISOString(),
        ];
    }
    
    // ส่งเฉพาะเมื่อ status เปลี่ยน
    public function broadcastWhen(): bool
    {
        return $this->order->wasChanged('status');
    }
}
```

```php
// ShouldBroadcastNow - ส่งทันทีโดยไม่ผ่าน Queue
class CriticalAlert implements ShouldBroadcastNow
{
    use Dispatchable, InteractsWithSockets, SerializesModels;
    
    public function __construct(
        public readonly string $message,
        public readonly string $severity
    ) {}
    
    public function broadcastOn(): array
    {
        return [new Channel('alerts')];
    }
}
```

### 5.2 Broadcasting จาก Controller

```php
// app/Http/Controllers/OrderController.php
class OrderController extends Controller
{
    public function updateStatus(Request $request, Order $order)
    {
        $validated = $request->validate([
            'status' => 'required|in:pending,processing,shipped,delivered,cancelled',
        ]);
        
        $order->update(['status' => $validated['status']]);
        
        // Broadcast event
        event(new OrderStatusUpdated($order));
        
        // หรือใช้ broadcast() helper
        broadcast(new OrderStatusUpdated($order))->toOthers();
        
        return response()->json(['message' => 'Order status updated']);
    }
}
```

### 5.3 Client Events

Client Events ช่วยให้ Client ส่ง Event ไปยัง Client อื่นๆ โดยตรง (Peer-to-peer ผ่าน Server)

```php
// Enable Client Events ใน config/broadcasting.php
'pusher' => [
    'options' => [
        'enabledTransports' => ['ws', 'wss'],
    ],
],
```

```javascript
// ส่ง Client Event (ชื่อต้องขึ้นต้นด้วย "client-")
Echo.private(`chat.${chatId}`)
    .whisper('typing', {
        user: currentUser,
    });

// รับ Client Event
Echo.private(`chat.${chatId}`)
    .listenForWhisper('typing', (event) => {
        showTypingIndicator(event.user);
        
        // ซ่อนหลัง 3 วินาที
        setTimeout(() => hideTypingIndicator(event.user), 3000);
    });
```

---

## 6. Broadcasting กับ Queue

### 6.1 ตั้งค่า Queue Connection

```bash
# .env
QUEUE_CONNECTION=redis
```

```php
// config/queue.php
'redis' => [
    'driver' => 'redis',
    'connection' => 'default',
    'queue' => env('REDIS_QUEUE', 'default'),
    'retry_after' => 90,
    'block_for' => null,
    'after_commit' => false,
],
```

### 6.2 กำหนด Queue สำหรับ Broadcasting

```php
class OrderStatusUpdated implements ShouldBroadcast
{
    // ใช้ Queue Connection และ Queue Name เฉพาะ
    public $connection = 'redis';
    public $queue = 'broadcasting';
    
    // ...
}
```

### 6.3 รัน Queue Worker

```bash
# รัน worker สำหรับ broadcasting queue
php artisan queue:work redis --queue=broadcasting,default

# รันด้วย Supervisor
[program:laravel-broadcasting-worker]
command=php /var/www/html/artisan queue:work redis --queue=broadcasting --sleep=3 --tries=3
autostart=true
autorestart=true
numprocs=2
```

---

## 7. Workshop: Real-time Chat ด้วย Laravel + Echo

### Project Structure

```
chat-app/
├── app/
│   ├── Events/
│   │   ├── MessageSent.php
│   │   ├── UserTyping.php
│   │   └── UserOnlineStatus.php
│   ├── Models/
│   │   ├── Chat.php
│   │   ├── ChatMember.php
│   │   └── Message.php
│   └── Http/
│       └── Controllers/
│           ├── ChatController.php
│           └── MessageController.php
├── resources/
│   └── js/
│       └── chat/
│           ├── ChatApp.vue
│           ├── ChatRoom.vue
│           └── MessageInput.vue
└── routes/
    ├── channels.php
    └── api.php
```

### Step 1: สร้าง Models และ Migrations

```php
// database/migrations/create_chats_table.php
Schema::create('chats', function (Blueprint $table) {
    $table->id();
    $table->string('name')->nullable();
    $table->enum('type', ['direct', 'group'])->default('direct');
    $table->timestamps();
});

Schema::create('chat_members', function (Blueprint $table) {
    $table->id();
    $table->foreignId('chat_id')->constrained()->onDelete('cascade');
    $table->foreignId('user_id')->constrained()->onDelete('cascade');
    $table->enum('role', ['admin', 'member'])->default('member');
    $table->timestamp('last_read_at')->nullable();
    $table->timestamps();
    
    $table->unique(['chat_id', 'user_id']);
});

Schema::create('messages', function (Blueprint $table) {
    $table->id();
    $table->foreignId('chat_id')->constrained()->onDelete('cascade');
    $table->foreignId('user_id')->constrained()->onDelete('cascade');
    $table->text('content');
    $table->string('type')->default('text'); // text, image, file
    $table->json('attachments')->nullable();
    $table->timestamp('read_at')->nullable();
    $table->timestamps();
    $table->softDeletes();
});
```

```php
// app/Models/Chat.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Chat extends Model
{
    protected $fillable = ['name', 'type'];
    
    public function members(): BelongsToMany
    {
        return $this->belongsToMany(User::class, 'chat_members')
            ->withPivot(['role', 'last_read_at'])
            ->withTimestamps();
    }
    
    public function messages(): HasMany
    {
        return $this->hasMany(Message::class)->latest();
    }
    
    public function lastMessage(): HasOne
    {
        return $this->hasOne(Message::class)->latest();
    }
    
    public function unreadCount(int $userId): int
    {
        $member = $this->members()->where('user_id', $userId)->first();
        
        if (!$member) return 0;
        
        return $this->messages()
            ->where('user_id', '!=', $userId)
            ->where('created_at', '>', $member->pivot->last_read_at ?? '1970-01-01')
            ->count();
    }
}
```

```php
// app/Models/Message.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\SoftDeletes;

class Message extends Model
{
    use SoftDeletes;
    
    protected $fillable = ['chat_id', 'user_id', 'content', 'type', 'attachments'];
    
    protected $casts = [
        'attachments' => 'array',
        'read_at' => 'datetime',
    ];
    
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
    
    public function chat(): BelongsTo
    {
        return $this->belongsTo(Chat::class);
    }
}
```

### Step 2: สร้าง Events

```php
// app/Events/MessageSent.php
namespace App\Events;

use App\Models\Message;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class MessageSent implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;
    
    public function __construct(
        public readonly Message $message
    ) {}
    
    public function broadcastOn(): array
    {
        return [
            new PresenceChannel('chat.' . $this->message->chat_id),
        ];
    }
    
    public function broadcastAs(): string
    {
        return 'message.sent';
    }
    
    public function broadcastWith(): array
    {
        return [
            'id' => $this->message->id,
            'chat_id' => $this->message->chat_id,
            'content' => $this->message->content,
            'type' => $this->message->type,
            'attachments' => $this->message->attachments,
            'user' => [
                'id' => $this->message->user->id,
                'name' => $this->message->user->name,
                'avatar' => $this->message->user->avatar_url,
            ],
            'created_at' => $this->message->created_at->toISOString(),
        ];
    }
}
```

```php
// app/Events/UserTyping.php
namespace App\Events;

use App\Models\User;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class UserTyping implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;
    
    public function __construct(
        public readonly User $user,
        public readonly int $chatId,
        public readonly bool $isTyping
    ) {}
    
    public function broadcastOn(): array
    {
        return [
            new PresenceChannel('chat.' . $this->chatId),
        ];
    }
    
    public function broadcastAs(): string
    {
        return 'user.typing';
    }
    
    public function broadcastWith(): array
    {
        return [
            'user_id' => $this->user->id,
            'user_name' => $this->user->name,
            'is_typing' => $this->isTyping,
        ];
    }
    
    // ไม่ต้องผ่าน Queue - ส่งทันที
    public function broadcastWhen(): bool
    {
        return true;
    }
}
```

### Step 3: สร้าง Controllers

```php
// app/Http/Controllers/ChatController.php
namespace App\Http\Controllers;

use App\Models\Chat;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class ChatController extends Controller
{
    public function index(): JsonResponse
    {
        $chats = auth()->user()
            ->chats()
            ->with(['lastMessage.user', 'members'])
            ->withCount(['messages as unread_count' => function ($query) {
                $query->where('user_id', '!=', auth()->id())
                    ->whereNull('read_at');
            }])
            ->orderByDesc(function ($query) {
                $query->select('created_at')
                    ->from('messages')
                    ->whereColumn('chat_id', 'chats.id')
                    ->latest()
                    ->limit(1);
            })
            ->get();
        
        return response()->json($chats);
    }
    
    public function create(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'user_ids' => 'required|array|min:1',
            'user_ids.*' => 'exists:users,id',
            'name' => 'nullable|string|max:100',
        ]);
        
        $userIds = array_unique(array_merge(
            $validated['user_ids'],
            [auth()->id()]
        ));
        
        // ตรวจสอบว่ามี Direct Chat อยู่แล้วหรือไม่
        if (count($userIds) === 2) {
            $existingChat = $this->findDirectChat($userIds);
            if ($existingChat) {
                return response()->json($existingChat);
            }
        }
        
        $chat = Chat::create([
            'type' => count($userIds) > 2 ? 'group' : 'direct',
            'name' => $validated['name'] ?? null,
        ]);
        
        foreach ($userIds as $userId) {
            $chat->members()->attach($userId, [
                'role' => $userId === auth()->id() ? 'admin' : 'member',
            ]);
        }
        
        return response()->json($chat->load('members'), 201);
    }
    
    private function findDirectChat(array $userIds): ?Chat
    {
        return Chat::where('type', 'direct')
            ->whereHas('members', fn($q) => $q->where('user_id', $userIds[0]))
            ->whereHas('members', fn($q) => $q->where('user_id', $userIds[1]))
            ->first();
    }
}
```

```php
// app/Http/Controllers/MessageController.php
namespace App\Http\Controllers;

use App\Events\MessageSent;
use App\Events\UserTyping;
use App\Models\Chat;
use App\Models\Message;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class MessageController extends Controller
{
    public function index(Chat $chat): JsonResponse
    {
        $this->authorize('view', $chat);
        
        $messages = $chat->messages()
            ->with('user')
            ->latest()
            ->paginate(50);
        
        // Mark messages as read
        Message::where('chat_id', $chat->id)
            ->where('user_id', '!=', auth()->id())
            ->whereNull('read_at')
            ->update(['read_at' => now()]);
        
        // Update last read
        $chat->members()
            ->where('user_id', auth()->id())
            ->update(['last_read_at' => now()]);
        
        return response()->json($messages);
    }
    
    public function store(Request $request, Chat $chat): JsonResponse
    {
        $this->authorize('sendMessage', $chat);
        
        $validated = $request->validate([
            'content' => 'required_without:attachments|string|max:5000',
            'type' => 'nullable|in:text,image,file',
            'attachments' => 'nullable|array',
        ]);
        
        $message = $chat->messages()->create([
            'user_id' => auth()->id(),
            'content' => $validated['content'] ?? '',
            'type' => $validated['type'] ?? 'text',
            'attachments' => $validated['attachments'] ?? null,
        ]);
        
        $message->load('user');
        
        // Broadcast message
        broadcast(new MessageSent($message))->toOthers();
        
        return response()->json($message, 201);
    }
    
    public function typing(Request $request, Chat $chat): JsonResponse
    {
        $this->authorize('view', $chat);
        
        $isTyping = $request->boolean('is_typing');
        
        // Broadcast typing status
        broadcast(new UserTyping(auth()->user(), $chat->id, $isTyping))->toOthers();
        
        return response()->json(['status' => 'ok']);
    }
    
    public function destroy(Message $message): JsonResponse
    {
        $this->authorize('delete', $message);
        
        $message->delete();
        
        // Broadcast message deletion
        broadcast(new \App\Events\MessageDeleted($message))->toOthers();
        
        return response()->noContent();
    }
}
```

### Step 4: ตั้งค่า Channels Authorization

```php
// routes/channels.php
use App\Models\Chat;
use App\Models\User;

Broadcast::channel('App.Models.User.{id}', function ($user, $id) {
    return (int) $user->id === (int) $id;
});

// Chat Presence Channel
Broadcast::channel('chat.{chatId}', function (User $user, int $chatId) {
    $chat = Chat::find($chatId);
    
    if (!$chat) {
        return false;
    }
    
    $member = $chat->members()->where('user_id', $user->id)->first();
    
    if (!$member) {
        return false;
    }
    
    return [
        'id' => $user->id,
        'name' => $user->name,
        'avatar' => $user->avatar_url,
        'role' => $member->pivot->role,
    ];
});

// Personal Notifications Channel
Broadcast::channel('notifications.{userId}', function (User $user, int $userId) {
    return (int) $user->id === (int) $userId;
});
```

### Step 5: Vue.js Component

```vue
<!-- resources/js/components/ChatRoom.vue -->
<template>
  <div class="chat-room">
    <!-- Online Users -->
    <div class="online-users">
      <h3>Online ({{ onlineUsers.length }})</h3>
      <div v-for="user in onlineUsers" :key="user.id" class="user-avatar">
        <img :src="user.avatar" :alt="user.name" />
        <span>{{ user.name }}</span>
      </div>
    </div>
    
    <!-- Messages -->
    <div class="messages" ref="messagesContainer">
      <div
        v-for="message in messages"
        :key="message.id"
        :class="['message', { 'own': message.user.id === currentUser.id }]"
      >
        <img :src="message.user.avatar" :alt="message.user.name" />
        <div class="message-content">
          <span class="username">{{ message.user.name }}</span>
          <p>{{ message.content }}</p>
          <span class="time">{{ formatTime(message.created_at) }}</span>
        </div>
      </div>
      
      <!-- Typing Indicators -->
      <div v-if="typingUsers.length > 0" class="typing-indicator">
        <span>{{ typingText }}</span>
        <div class="dots">
          <span></span><span></span><span></span>
        </div>
      </div>
    </div>
    
    <!-- Input -->
    <div class="message-input">
      <textarea
        v-model="newMessage"
        @keydown.enter.prevent="sendMessage"
        @input="handleTyping"
        placeholder="Type a message..."
      ></textarea>
      <button @click="sendMessage" :disabled="!newMessage.trim()">Send</button>
    </div>
  </div>
</template>

<script>
export default {
  props: {
    chatId: Number,
    currentUser: Object,
  },
  
  data() {
    return {
      messages: [],
      newMessage: '',
      onlineUsers: [],
      typingUsers: [],
      typingTimeout: null,
      isTyping: false,
      channel: null,
    };
  },
  
  computed: {
    typingText() {
      if (this.typingUsers.length === 0) return '';
      if (this.typingUsers.length === 1) return `${this.typingUsers[0].user_name} is typing...`;
      return `${this.typingUsers.length} people are typing...`;
    },
  },
  
  async mounted() {
    await this.loadMessages();
    this.subscribeToChannel();
  },
  
  beforeUnmount() {
    Echo.leave(`chat.${this.chatId}`);
  },
  
  methods: {
    async loadMessages() {
      const response = await fetch(`/api/chats/${this.chatId}/messages`);
      const data = await response.json();
      this.messages = data.data.reverse();
      this.$nextTick(() => this.scrollToBottom());
    },
    
    subscribeToChannel() {
      this.channel = Echo.join(`chat.${this.chatId}`)
        .here((users) => {
          this.onlineUsers = users;
        })
        .joining((user) => {
          this.onlineUsers.push(user);
        })
        .leaving((user) => {
          this.onlineUsers = this.onlineUsers.filter(u => u.id !== user.id);
        })
        .listen('.message.sent', (event) => {
          this.messages.push(event);
          this.$nextTick(() => this.scrollToBottom());
        })
        .listen('.user.typing', (event) => {
          this.handleTypingEvent(event);
        });
    },
    
    async sendMessage() {
      if (!this.newMessage.trim()) return;
      
      const message = this.newMessage;
      this.newMessage = '';
      
      try {
        const response = await fetch(`/api/chats/${this.chatId}/messages`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]').content,
          },
          body: JSON.stringify({ content: message }),
        });
        
        const data = await response.json();
        this.messages.push(data);
        this.$nextTick(() => this.scrollToBottom());
        
        // Stop typing indicator
        this.stopTyping();
      } catch (error) {
        console.error('Failed to send message:', error);
        this.newMessage = message;
      }
    },
    
    handleTyping() {
      if (!this.isTyping) {
        this.isTyping = true;
        this.broadcastTyping(true);
      }
      
      clearTimeout(this.typingTimeout);
      
      this.typingTimeout = setTimeout(() => {
        this.stopTyping();
      }, 2000);
    },
    
    stopTyping() {
      if (this.isTyping) {
        this.isTyping = false;
        this.broadcastTyping(false);
      }
      
      clearTimeout(this.typingTimeout);
    },
    
    async broadcastTyping(isTyping) {
      await fetch(`/api/chats/${this.chatId}/typing`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]').content,
        },
        body: JSON.stringify({ is_typing: isTyping }),
      });
    },
    
    handleTypingEvent(event) {
      if (event.user_id === this.currentUser.id) return;
      
      if (event.is_typing) {
        if (!this.typingUsers.find(u => u.user_id === event.user_id)) {
          this.typingUsers.push(event);
        }
      } else {
        this.typingUsers = this.typingUsers.filter(u => u.user_id !== event.user_id);
      }
    },
    
    scrollToBottom() {
      const container = this.$refs.messagesContainer;
      if (container) {
        container.scrollTop = container.scrollHeight;
      }
    },
    
    formatTime(dateString) {
      return new Date(dateString).toLocaleTimeString('th-TH', {
        hour: '2-digit',
        minute: '2-digit',
      });
    },
  },
};
</script>
```

### Step 6: สร้าง Notifications System

```php
// app/Notifications/NewMessageNotification.php
namespace App\Notifications;

use App\Models\Message;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Messages\BroadcastMessage;
use Illuminate\Notifications\Notification;

class NewMessageNotification extends Notification implements ShouldQueue
{
    use Queueable;
    
    public function __construct(
        private readonly Message $message
    ) {}
    
    public function via($notifiable): array
    {
        return ['broadcast', 'database'];
    }
    
    public function toBroadcast($notifiable): BroadcastMessage
    {
        return new BroadcastMessage([
            'type' => 'new_message',
            'message' => [
                'id' => $this->message->id,
                'content' => substr($this->message->content, 0, 100),
                'sender' => $this->message->user->name,
                'chat_id' => $this->message->chat_id,
            ],
            'unread_count' => $notifiable->unreadMessagesCount(),
        ]);
    }
    
    public function toArray($notifiable): array
    {
        return [
            'type' => 'new_message',
            'message_id' => $this->message->id,
            'chat_id' => $this->message->chat_id,
            'sender_name' => $this->message->user->name,
        ];
    }
    
    public function broadcastOn(): array
    {
        return [
            new PrivateChannel('App.Models.User.' . $this->message->user_id),
        ];
    }
}
```

```javascript
// รับ Notification ใน JavaScript
Echo.private(`App.Models.User.${currentUser.id}`)
    .notification((notification) => {
        if (notification.type === 'new_message') {
            showNotificationBadge(notification.unread_count);
            
            if (Notification.permission === 'granted') {
                new Notification(`New message from ${notification.message.sender}`, {
                    body: notification.message.content,
                    icon: '/favicon.ico',
                });
            }
        }
    });
```

---

## 8. Performance Considerations

### 8.1 Connection Scaling

```bash
# Soketi ใช้ Redis สำหรับ Horizontal Scaling
SOKETI_DB_DRIVER=redis
SOKETI_DB_REDIS_HOST=redis
SOKETI_DB_REDIS_PORT=6379
SOKETI_DB_REDIS_DB=0
```

### 8.2 Message Throttling

```php
// app/Http/Controllers/MessageController.php
public function store(Request $request, Chat $chat)
{
    // Rate limit: 10 messages per minute per user per chat
    $key = 'message:' . auth()->id() . ':chat:' . $chat->id;
    
    if (RateLimiter::tooManyAttempts($key, 10)) {
        return response()->json([
            'error' => 'Too many messages. Please slow down.',
        ], 429);
    }
    
    RateLimiter::hit($key, 60);
    
    // ...
}
```

---

## Quiz

**ข้อ 1:** Presence Channel ต่างจาก Private Channel อย่างไร?

a) Presence Channel ไม่ต้องการ Authentication  
b) Presence Channel ติดตาม Online Users ได้ ส่วน Private Channel ไม่ได้  
c) Private Channel เร็วกว่า Presence Channel  
d) ไม่มีความแตกต่าง

**เฉลย:** b) Presence Channel มีความสามารถพิเศษในการติดตามว่ามี Users ใดอยู่ใน Channel บ้าง ผ่าน `here()`, `joining()`, และ `leaving()` callbacks

---

**ข้อ 2:** `broadcast()->toOthers()` ต่างจาก `broadcast()` อย่างไร?

a) `toOthers()` ส่งไปยังทุก Client รวมถึง Sender  
b) `toOthers()` ส่งไปยังทุก Client ยกเว้น Sender  
c) `toOthers()` ส่งไปยัง Sender เท่านั้น  
d) ไม่มีความแตกต่าง

**เฉลย:** b) `broadcast()->toOthers()` จะไม่ส่ง Event กลับไปยัง Client ที่ trigger event นั้น ทำให้ไม่เกิด duplicate ในฝั่ง Client

---

**ข้อ 3:** ควรใช้ `ShouldBroadcast` หรือ `ShouldBroadcastNow`?

a) ใช้ `ShouldBroadcastNow` เสมอเพราะเร็วกว่า  
b) ใช้ `ShouldBroadcast` สำหรับ non-critical events และ `ShouldBroadcastNow` สำหรับ critical events  
c) ใช้ `ShouldBroadcast` เสมอ  
d) ขึ้นอยู่กับ Database Driver

**เฉลย:** b) `ShouldBroadcast` ส่งผ่าน Queue ดีสำหรับ performance, `ShouldBroadcastNow` ส่งทันทีเหมาะกับ time-critical events เช่น Alerts

---

## สรุป

ใน Part นี้คุณได้เรียนรู้:
- **Echo.js Setup:** การตั้งค่า Laravel Echo กับ Soketi
- **Channel Types:** Public, Private, Presence Channels
- **Broadcasting Events:** ShouldBroadcast, broadcastWith, broadcastWhen
- **Workshop:** Real-time Chat Application ครบวงจร

---

## ไปต่อ

➡️ [Part 045: Laravel Livewire — Reactive UI โดยไม่ต้องเขียน JavaScript](./part-045-laravel-livewire.md)
