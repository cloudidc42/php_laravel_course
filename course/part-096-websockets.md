# Part 96: WebSockets ด้วย PHP/Laravel

## บทนำ

WebSocket คือ Protocol ที่ให้ Two-way Communication ระหว่าง Client และ Server แบบ Real-time แตกต่างจาก HTTP ที่เป็น Request-Response

**ใช้เมื่อ**:
- Chat Applications
- Real-time Notifications
- Live Dashboards
- Collaborative Editing
- Online Games
- Live Trading Systems

---

## WebSocket ด้วย PHP (Ratchet)

### Installation

```bash
composer require cboden/ratchet
```

### Basic WebSocket Server

```php
<?php

use Ratchet\MessageComponentInterface;
use Ratchet\ConnectionInterface;
use Ratchet\Server\IoServer;
use Ratchet\Http\HttpServer;
use Ratchet\WebSocket\WsServer;

class ChatServer implements MessageComponentInterface
{
    private \SplObjectStorage $clients;
    private array $rooms = []; // roomId => [connections]
    private array $userConnections = []; // userId => connection
    
    public function __construct()
    {
        $this->clients = new \SplObjectStorage;
        echo "Chat server started\n";
    }
    
    public function onOpen(ConnectionInterface $conn): void
    {
        $this->clients->attach($conn);
        
        $conn->userData = [
            'id' => $conn->resourceId,
            'user_id' => null,
            'username' => null,
            'rooms' => [],
        ];
        
        echo "New connection: {$conn->resourceId}\n";
    }
    
    public function onMessage(ConnectionInterface $from, $msg): void
    {
        $data = json_decode($msg, true);
        
        if (!$data || !isset($data['type'])) {
            $from->send(json_encode(['error' => 'Invalid message format']));
            return;
        }
        
        switch ($data['type']) {
            case 'auth':
                $this->handleAuth($from, $data);
                break;
                
            case 'join_room':
                $this->handleJoinRoom($from, $data);
                break;
                
            case 'leave_room':
                $this->handleLeaveRoom($from, $data);
                break;
                
            case 'message':
                $this->handleMessage($from, $data);
                break;
                
            case 'typing':
                $this->handleTyping($from, $data);
                break;
                
            default:
                $from->send(json_encode(['error' => "Unknown message type: {$data['type']}"]));
        }
    }
    
    private function handleAuth(ConnectionInterface $conn, array $data): void
    {
        // Verify JWT token
        $token = $data['token'] ?? '';
        $userId = $this->verifyToken($token);
        
        if (!$userId) {
            $conn->send(json_encode(['type' => 'error', 'message' => 'Invalid token']));
            $conn->close();
            return;
        }
        
        $conn->userData['user_id'] = $userId;
        $conn->userData['username'] = $this->getUserName($userId);
        $this->userConnections[$userId] = $conn;
        
        $conn->send(json_encode([
            'type' => 'authenticated',
            'user_id' => $userId,
            'username' => $conn->userData['username'],
        ]));
        
        echo "User authenticated: {$userId}\n";
    }
    
    private function handleJoinRoom(ConnectionInterface $conn, array $data): void
    {
        $roomId = $data['room_id'];
        
        if (!isset($this->rooms[$roomId])) {
            $this->rooms[$roomId] = new \SplObjectStorage;
        }
        
        $this->rooms[$roomId]->attach($conn);
        $conn->userData['rooms'][] = $roomId;
        
        // ส่ง history ล่าสุด
        $history = $this->getMessageHistory($roomId);
        
        $conn->send(json_encode([
            'type' => 'room_joined',
            'room_id' => $roomId,
            'history' => $history,
        ]));
        
        // แจ้ง Members ในห้อง
        $this->broadcastToRoom($roomId, [
            'type' => 'user_joined',
            'room_id' => $roomId,
            'user_id' => $conn->userData['user_id'],
            'username' => $conn->userData['username'],
        ], except: $conn);
    }
    
    private function handleMessage(ConnectionInterface $from, array $data): void
    {
        if (!$from->userData['user_id']) {
            $from->send(json_encode(['error' => 'Not authenticated']));
            return;
        }
        
        $roomId = $data['room_id'];
        $content = htmlspecialchars($data['content'], ENT_QUOTES, 'UTF-8');
        
        // Validate
        if (empty(trim($content))) return;
        if (strlen($content) > 2000) {
            $from->send(json_encode(['error' => 'Message too long']));
            return;
        }
        
        $message = [
            'type' => 'message',
            'id' => uniqid('msg_', true),
            'room_id' => $roomId,
            'user_id' => $from->userData['user_id'],
            'username' => $from->userData['username'],
            'content' => $content,
            'timestamp' => time(),
        ];
        
        // Save to database
        $this->saveMessage($message);
        
        // Broadcast to room
        $this->broadcastToRoom($roomId, $message);
    }
    
    private function handleTyping(ConnectionInterface $from, array $data): void
    {
        $roomId = $data['room_id'];
        
        $this->broadcastToRoom($roomId, [
            'type' => 'typing',
            'room_id' => $roomId,
            'user_id' => $from->userData['user_id'],
            'username' => $from->userData['username'],
            'is_typing' => $data['is_typing'] ?? true,
        ], except: $from);
    }
    
    public function onClose(ConnectionInterface $conn): void
    {
        $this->clients->detach($conn);
        
        $userId = $conn->userData['user_id'];
        if ($userId) {
            unset($this->userConnections[$userId]);
        }
        
        // Leave all rooms
        foreach ($conn->userData['rooms'] as $roomId) {
            if (isset($this->rooms[$roomId])) {
                $this->rooms[$roomId]->detach($conn);
                
                // Notify room
                $this->broadcastToRoom($roomId, [
                    'type' => 'user_left',
                    'room_id' => $roomId,
                    'user_id' => $userId,
                    'username' => $conn->userData['username'] ?? 'Unknown',
                ]);
            }
        }
        
        echo "Connection closed: {$conn->resourceId}\n";
    }
    
    public function onError(ConnectionInterface $conn, \Exception $e): void
    {
        echo "Error on {$conn->resourceId}: {$e->getMessage()}\n";
        $conn->close();
    }
    
    // Send to specific user
    public function sendToUser(int $userId, array $data): void
    {
        $conn = $this->userConnections[$userId] ?? null;
        $conn?->send(json_encode($data));
    }
    
    // Broadcast to all in room
    private function broadcastToRoom(
        string $roomId,
        array $data,
        ?ConnectionInterface $except = null
    ): void {
        if (!isset($this->rooms[$roomId])) return;
        
        foreach ($this->rooms[$roomId] as $client) {
            if ($except && $client === $except) continue;
            $client->send(json_encode($data));
        }
    }
    
    // Broadcast to all connections
    public function broadcastAll(array $data): void
    {
        foreach ($this->clients as $client) {
            $client->send(json_encode($data));
        }
    }
    
    private function verifyToken(string $token): ?int
    {
        // Verify JWT token and return user ID
        // ใช้ Library จริงๆ อย่าง firebase/php-jwt
        return 1; // Mock
    }
    
    private function getUserName(int $userId): string
    {
        // Fetch from database
        return "User {$userId}"; // Mock
    }
    
    private function getMessageHistory(string $roomId, int $limit = 50): array
    {
        // Fetch from database
        return [];
    }
    
    private function saveMessage(array $message): void
    {
        // Save to database
    }
}

// Start Server
$server = IoServer::factory(
    new HttpServer(
        new WsServer(
            new ChatServer()
        )
    ),
    8080
);

echo "WebSocket server running on ws://localhost:8080\n";
$server->run();
```

---

## Laravel Broadcasting

### Setup

```bash
# Install Pusher or Soketi
composer require pusher/pusher-php-server

# .env
BROADCAST_DRIVER=pusher
PUSHER_APP_ID=your-app-id
PUSHER_APP_KEY=your-app-key
PUSHER_APP_SECRET=your-app-secret
PUSHER_APP_CLUSTER=mt1
PUSHER_SCHEME=https
PUSHER_PORT=443
```

### Events

```php
<?php

namespace App\Events;

use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Contracts\Broadcasting\ShouldBroadcastNow;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

// Public Channel Event
class NewProductLaunched implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;
    
    public function __construct(
        public readonly array $product
    ) {}
    
    public function broadcastOn(): array
    {
        return [new Channel('products')];
    }
    
    public function broadcastAs(): string
    {
        return 'product.launched';
    }
    
    public function broadcastWith(): array
    {
        return [
            'id' => $this->product['id'],
            'name' => $this->product['name'],
            'price' => $this->product['price'],
            'image' => $this->product['main_image'],
        ];
    }
}

// Private Channel Event (ต้องการ Auth)
class OrderStatusUpdated implements ShouldBroadcastNow
{
    use Dispatchable, InteractsWithSockets, SerializesModels;
    
    public function __construct(
        private readonly \App\Models\Order $order
    ) {}
    
    public function broadcastOn(): PrivateChannel
    {
        return new PrivateChannel('orders.' . $this->order->user_id);
    }
    
    public function broadcastWith(): array
    {
        return [
            'order_id' => $this->order->id,
            'status' => $this->order->status,
            'updated_at' => $this->order->updated_at->toISOString(),
        ];
    }
}

// Presence Channel Event (รู้ว่า User ไหนออนไลน์)
class UserJoinedRoom implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;
    
    public function __construct(
        private readonly int $roomId,
        private readonly array $user
    ) {}
    
    public function broadcastOn(): PresenceChannel
    {
        return new PresenceChannel('chat-room.' . $this->roomId);
    }
    
    public function broadcastWith(): array
    {
        return [
            'user_id' => $this->user['id'],
            'username' => $this->user['name'],
            'avatar' => $this->user['avatar'],
        ];
    }
}

// ใช้งาน
broadcast(new OrderStatusUpdated($order))->toOthers();

// หรือ
event(new NewProductLaunched($product->toArray()));
```

### Channel Authorization

```php
<?php

// routes/channels.php
use Illuminate\Support\Facades\Broadcast;
use App\Models\{User, Order, Room};

// Private Channel - ต้องเป็นเจ้าของ Order
Broadcast::channel('orders.{userId}', function (User $user, int $userId) {
    return $user->id === $userId;
});

// Presence Channel - ต้องเป็น Member ของ Room
Broadcast::channel('chat-room.{roomId}', function (User $user, int $roomId) {
    $room = Room::find($roomId);
    
    if (!$room || !$room->members->contains($user->id)) {
        return false;
    }
    
    // Return user data ที่จะแชร์ใน Presence Channel
    return [
        'id' => $user->id,
        'name' => $user->name,
        'avatar' => $user->avatar_url,
        'status' => 'online',
    ];
});

// Admin Channel
Broadcast::channel('admin.dashboard', function (User $user) {
    return $user->isAdmin();
});
```

---

## Soketi (Self-hosted Pusher)

```bash
# ติดตั้ง Soketi
npm install -g @soketi/soketi

# หรือ Docker
docker run -p 6001:6001 quay.io/soketi/soketi:latest

# .env สำหรับ Soketi
PUSHER_APP_ID=app-id
PUSHER_APP_KEY=app-key
PUSHER_APP_SECRET=app-secret
PUSHER_HOST=localhost
PUSHER_PORT=6001
PUSHER_SCHEME=http
```

```yaml
# soketi-config.json
{
    "debug": false,
    "port": 6001,
    "appManager.driver": "array",
    "appManager.array.apps": [
        {
            "id": "app-id",
            "key": "app-key",
            "secret": "app-secret",
            "webhooks": []
        }
    ],
    "database.driver": "redis",
    "databasePooling.enabled": true
}
```

---

## Real-time Features

### Live Notifications

```php
<?php

namespace App\Services;

class NotificationBroadcaster
{
    public function sendToUser(int $userId, array $notification): void
    {
        broadcast(new UserNotification($userId, $notification));
        
        // ยัง Save ไปในฐานข้อมูลด้วย
        \DB::table('notifications')->insert([
            'user_id' => $userId,
            'type' => $notification['type'],
            'data' => json_encode($notification['data']),
            'created_at' => now(),
        ]);
    }
    
    public function sendToAll(array $notification): void
    {
        broadcast(new SystemNotification($notification));
    }
}

// ใช้ใน Service
class OrderService
{
    public function __construct(
        private NotificationBroadcaster $broadcaster
    ) {}
    
    public function updateStatus(Order $order, string $newStatus): void
    {
        $order->update(['status' => $newStatus]);
        
        $this->broadcaster->sendToUser($order->user_id, [
            'type' => 'order_status_updated',
            'data' => [
                'order_id' => $order->id,
                'old_status' => $order->getOriginal('status'),
                'new_status' => $newStatus,
                'message' => "คำสั่งซื้อ #{$order->id} ถูกอัปเดตเป็น {$newStatus}",
            ],
        ]);
    }
}
```

### Live Dashboard

```php
<?php

namespace App\Events;

class DashboardStatsUpdated implements ShouldBroadcastNow
{
    use Dispatchable, InteractsWithSockets;
    
    public function broadcastOn(): Channel
    {
        return new Channel('admin.dashboard');
    }
    
    public function broadcastWith(): array
    {
        return [
            'total_orders_today' => Order::today()->count(),
            'revenue_today' => Order::today()->sum('total'),
            'active_users' => User::activeToday()->count(),
            'pending_orders' => Order::pending()->count(),
            'updated_at' => now()->toISOString(),
        ];
    }
}

// Schedule ส่งทุก 30 วินาที
// routes/console.php
Schedule::call(function () {
    broadcast(new DashboardStatsUpdated());
})->everyThirtySeconds();
```

---

## Workshop: Chat Application

### Backend

```php
<?php

// Models
class ChatRoom extends Model
{
    protected $fillable = ['name', 'type', 'created_by'];
    
    public function members(): BelongsToMany
    {
        return $this->belongsToMany(User::class, 'chat_room_members')
            ->withPivot(['role', 'joined_at', 'last_read_at'])
            ->withTimestamps();
    }
    
    public function messages(): HasMany
    {
        return $this->hasMany(ChatMessage::class, 'room_id');
    }
    
    public function latestMessage(): HasOne
    {
        return $this->hasOne(ChatMessage::class, 'room_id')->latest();
    }
}

class ChatMessage extends Model
{
    protected $fillable = ['room_id', 'user_id', 'content', 'type', 'metadata'];
    
    protected $casts = [
        'metadata' => 'array',
    ];
    
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
    
    public function room(): BelongsTo
    {
        return $this->belongsTo(ChatRoom::class, 'room_id');
    }
    
    public function readBy(): BelongsToMany
    {
        return $this->belongsToMany(User::class, 'message_reads');
    }
}

// Controllers
class ChatController extends Controller
{
    public function index(): JsonResponse
    {
        $rooms = ChatRoom::whereHas('members', function ($q) {
            $q->where('user_id', auth()->id());
        })->with([
            'latestMessage.user',
            'members' => fn($q) => $q->limit(5)
        ])->get();
        
        return response()->json($rooms);
    }
    
    public function messages(ChatRoom $room): JsonResponse
    {
        $this->authorize('view', $room);
        
        $messages = $room->messages()
            ->with('user')
            ->latest()
            ->paginate(50);
        
        // Mark messages as read
        $this->markAsRead($room);
        
        return response()->json($messages);
    }
    
    public function sendMessage(Request $request, ChatRoom $room): JsonResponse
    {
        $this->authorize('message', $room);
        
        $validated = $request->validate([
            'content' => 'required|string|max:2000',
            'type' => 'in:text,image,file',
        ]);
        
        $message = $room->messages()->create([
            'user_id' => auth()->id(),
            'content' => $validated['content'],
            'type' => $validated['type'] ?? 'text',
        ]);
        
        $message->load('user');
        
        // Broadcast
        broadcast(new NewChatMessage($room->id, $message))->toOthers();
        
        return response()->json($message, 201);
    }
    
    public function typing(Request $request, ChatRoom $room): JsonResponse
    {
        broadcast(new UserTyping(
            $room->id,
            auth()->user(),
            $request->boolean('is_typing')
        ))->toOthers();
        
        return response()->json(['success' => true]);
    }
    
    private function markAsRead(ChatRoom $room): void
    {
        \DB::table('chat_room_members')
            ->where('room_id', $room->id)
            ->where('user_id', auth()->id())
            ->update(['last_read_at' => now()]);
    }
}

// Events
class NewChatMessage implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;
    
    public function __construct(
        private readonly int $roomId,
        private readonly ChatMessage $message
    ) {}
    
    public function broadcastOn(): PresenceChannel
    {
        return new PresenceChannel('chat-room.' . $this->roomId);
    }
    
    public function broadcastAs(): string
    {
        return 'new.message';
    }
    
    public function broadcastWith(): array
    {
        return [
            'message' => [
                'id' => $this->message->id,
                'content' => $this->message->content,
                'type' => $this->message->type,
                'user' => [
                    'id' => $this->message->user->id,
                    'name' => $this->message->user->name,
                    'avatar' => $this->message->user->avatar_url,
                ],
                'created_at' => $this->message->created_at->toISOString(),
            ]
        ];
    }
}

class UserTyping implements ShouldBroadcastNow
{
    use Dispatchable, InteractsWithSockets;
    
    public function __construct(
        private readonly int $roomId,
        private readonly User $user,
        private readonly bool $isTyping
    ) {}
    
    public function broadcastOn(): PresenceChannel
    {
        return new PresenceChannel('chat-room.' . $this->roomId);
    }
    
    public function broadcastAs(): string
    {
        return 'user.typing';
    }
    
    public function broadcastWith(): array
    {
        return [
            'user_id' => $this->user->id,
            'username' => $this->user->name,
            'is_typing' => $this->isTyping,
        ];
    }
}
```

### Frontend (JavaScript)

```javascript
// chat.js - ใช้ Laravel Echo + Pusher.js

import Echo from 'laravel-echo';
import Pusher from 'pusher-js';

window.Pusher = Pusher;

const echo = new Echo({
    broadcaster: 'pusher',
    key: import.meta.env.VITE_PUSHER_APP_KEY,
    cluster: import.meta.env.VITE_PUSHER_APP_CLUSTER,
    forceTLS: true,
    
    // สำหรับ Soketi
    // wsHost: 'localhost',
    // wsPort: 6001,
    // forceTLS: false,
    // enabledTransports: ['ws', 'wss'],
});

class ChatApp {
    constructor(roomId) {
        this.roomId = roomId;
        this.channel = null;
        this.typingTimeout = null;
    }
    
    connect() {
        // Join Presence Channel
        this.channel = echo.join(`chat-room.${this.roomId}`)
            .here((users) => {
                console.log('Online users:', users);
                this.updateOnlineUsers(users);
            })
            .joining((user) => {
                console.log(`${user.name} joined`);
                this.addOnlineUser(user);
            })
            .leaving((user) => {
                console.log(`${user.name} left`);
                this.removeOnlineUser(user);
            })
            .listen('.new.message', (data) => {
                this.appendMessage(data.message);
            })
            .listen('.user.typing', (data) => {
                this.showTypingIndicator(data);
            })
            .error((error) => {
                console.error('Channel error:', error);
            });
    }
    
    async sendMessage(content) {
        try {
            const response = await fetch(`/api/chat/${this.roomId}/messages`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                    'Authorization': `Bearer ${this.getToken()}`,
                    'X-Socket-ID': echo.socketId(),
                },
                body: JSON.stringify({ content }),
            });
            
            const message = await response.json();
            this.appendMessage(message); // Optimistic update
        } catch (error) {
            console.error('Failed to send message:', error);
        }
    }
    
    onTyping() {
        // Notify typing
        fetch(`/api/chat/${this.roomId}/typing`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${this.getToken()}`,
            },
            body: JSON.stringify({ is_typing: true }),
        });
        
        // Stop typing after 3 seconds of inactivity
        clearTimeout(this.typingTimeout);
        this.typingTimeout = setTimeout(() => {
            fetch(`/api/chat/${this.roomId}/typing`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                    'Authorization': `Bearer ${this.getToken()}`,
                },
                body: JSON.stringify({ is_typing: false }),
            });
        }, 3000);
    }
    
    appendMessage(message) {
        const container = document.getElementById('messages');
        const div = document.createElement('div');
        div.className = 'message';
        div.innerHTML = `
            <img src="${message.user.avatar}" alt="${message.user.name}" class="avatar">
            <div class="content">
                <span class="username">${message.user.name}</span>
                <p>${message.content}</p>
                <span class="time">${new Date(message.created_at).toLocaleTimeString()}</span>
            </div>
        `;
        container.appendChild(div);
        container.scrollTop = container.scrollHeight;
    }
    
    showTypingIndicator({ user_id, username, is_typing }) {
        const indicator = document.getElementById(`typing-${user_id}`);
        
        if (is_typing) {
            if (!indicator) {
                const div = document.createElement('div');
                div.id = `typing-${user_id}`;
                div.className = 'typing-indicator';
                div.textContent = `${username} กำลังพิมพ์...`;
                document.getElementById('typing-container').appendChild(div);
            }
        } else {
            indicator?.remove();
        }
    }
    
    disconnect() {
        echo.leave(`chat-room.${this.roomId}`);
    }
    
    getToken() {
        return localStorage.getItem('auth_token');
    }
}

// Usage
const chat = new ChatApp(1);
chat.connect();

document.getElementById('message-input').addEventListener('keypress', (e) => {
    chat.onTyping();
    
    if (e.key === 'Enter' && !e.shiftKey) {
        e.preventDefault();
        const content = e.target.value.trim();
        if (content) {
            chat.sendMessage(content);
            e.target.value = '';
        }
    }
});
```

---

## สรุป

| Technology | Protocol | Latency | Use Case |
|-----------|---------|---------|---------|
| WebSocket (Ratchet) | WS | Very Low | Custom Server |
| Laravel Broadcasting | WS | Low | Laravel Apps |
| Pusher | WS over HTTPS | Low | Managed Service |
| Soketi | WS | Very Low | Self-hosted |
| Server-Sent Events | HTTP | Low | One-way Updates |
| Long Polling | HTTP | Medium | Simple Updates |

---

*WebSocket เปลี่ยน Web จาก "ถาม-ตอบ" เป็น "สนทนาแบบ Real-time"*
