# Part 049: Laravel Advanced Patterns — Clean Code ระดับมืออาชีพ

**ระดับ: ระดับโลก (World-class)**
**เวลาเรียน: 7-9 ชั่วโมง**

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ Repository Pattern เพื่อ Decouple Data Access
- สร้าง Service Layer ที่ชัดเจน
- ออกแบบ Action Classes แบบ Single-Responsibility
- ใช้ DTOs (Data Transfer Objects) เพื่อ Type Safety
- Refactor Laravel App ให้ Clean และ Maintainable

---

## 1. ทำไมต้องมี Advanced Patterns?

```php
// BAD: Controller ทำทุกอย่าง — Fat Controller Anti-pattern
class OrderController extends Controller
{
    public function store(Request $request)
    {
        // Validation
        $data = $request->validate([...]);
        
        // Business Logic
        $user = auth()->user();
        if ($user->credit < $data['total']) {
            return back()->withErrors(['credit' => 'Insufficient credit']);
        }
        
        // Database Operations
        $order = Order::create([...]);
        
        foreach ($data['items'] as $item) {
            $order->items()->create($item);
            Product::find($item['product_id'])->decrement('stock', $item['quantity']);
        }
        
        // External Services
        Mail::to($user)->send(new OrderConfirmation($order));
        $this->sendSmsNotification($user->phone, $order);
        $this->updateInventorySystem($order);
        
        return redirect()->route('orders.show', $order);
    }
}

// GOOD: Clean Architecture - แต่ละส่วนมีหน้าที่ชัดเจน
class OrderController extends Controller
{
    public function __construct(
        private readonly CreateOrderAction $createOrder
    ) {}
    
    public function store(StoreOrderRequest $request)
    {
        $order = ($this->createOrder)(
            CreateOrderData::fromRequest($request)
        );
        
        return redirect()->route('orders.show', $order);
    }
}
```

---

## 2. Repository Pattern

Repository เป็น Abstraction Layer ระหว่าง Business Logic และ Data Access

### 2.1 สร้าง Repository Interface

```php
// app/Contracts/Repositories/ProductRepositoryInterface.php
namespace App\Contracts\Repositories;

use App\Models\Product;
use Illuminate\Contracts\Pagination\LengthAwarePaginator;
use Illuminate\Database\Eloquent\Collection;

interface ProductRepositoryInterface
{
    public function findById(int $id): ?Product;
    
    public function findBySlug(string $slug): ?Product;
    
    public function getAll(array $filters = []): Collection;
    
    public function paginate(int $perPage = 15, array $filters = []): LengthAwarePaginator;
    
    public function create(array $data): Product;
    
    public function update(Product $product, array $data): Product;
    
    public function delete(Product $product): bool;
    
    public function getByCategory(int $categoryId, int $limit = 10): Collection;
    
    public function searchByName(string $query): Collection;
}
```

### 2.2 Implement Repository

```php
// app/Repositories/EloquentProductRepository.php
namespace App\Repositories;

use App\Contracts\Repositories\ProductRepositoryInterface;
use App\Models\Product;
use Illuminate\Contracts\Pagination\LengthAwarePaginator;
use Illuminate\Database\Eloquent\Collection;
use Illuminate\Support\Facades\Cache;

class EloquentProductRepository implements ProductRepositoryInterface
{
    public function findById(int $id): ?Product
    {
        return Cache::remember('product:' . $id, 3600, function () use ($id) {
            return Product::with(['category', 'images', 'specifications'])
                ->find($id);
        });
    }
    
    public function findBySlug(string $slug): ?Product
    {
        return Cache::remember('product:slug:' . $slug, 3600, function () use ($slug) {
            return Product::with(['category', 'images'])
                ->where('slug', $slug)
                ->first();
        });
    }
    
    public function getAll(array $filters = []): Collection
    {
        return $this->applyFilters(Product::query(), $filters)->get();
    }
    
    public function paginate(int $perPage = 15, array $filters = []): LengthAwarePaginator
    {
        return $this->applyFilters(Product::query(), $filters)->paginate($perPage);
    }
    
    public function create(array $data): Product
    {
        $product = Product::create($data);
        $this->clearCache();
        
        return $product;
    }
    
    public function update(Product $product, array $data): Product
    {
        $product->update($data);
        $this->clearProductCache($product->id, $product->slug);
        
        return $product->fresh();
    }
    
    public function delete(Product $product): bool
    {
        $this->clearProductCache($product->id, $product->slug);
        
        return $product->delete();
    }
    
    public function getByCategory(int $categoryId, int $limit = 10): Collection
    {
        return Cache::remember("products:category:{$categoryId}:{$limit}", 1800, function () use ($categoryId, $limit) {
            return Product::where('category_id', $categoryId)
                ->where('is_active', true)
                ->latest()
                ->limit($limit)
                ->get();
        });
    }
    
    public function searchByName(string $query): Collection
    {
        return Product::where('name', 'like', "%{$query}%")
            ->orWhere('description', 'like', "%{$query}%")
            ->where('is_active', true)
            ->limit(20)
            ->get();
    }
    
    private function applyFilters($query, array $filters)
    {
        return $query
            ->when(isset($filters['category_id']), fn($q) => 
                $q->where('category_id', $filters['category_id']))
            ->when(isset($filters['min_price']), fn($q) => 
                $q->where('price', '>=', $filters['min_price']))
            ->when(isset($filters['max_price']), fn($q) => 
                $q->where('price', '<=', $filters['max_price']))
            ->when(isset($filters['is_active']), fn($q) => 
                $q->where('is_active', $filters['is_active']))
            ->when(isset($filters['sort']), fn($q) => 
                $q->orderBy($filters['sort'], $filters['direction'] ?? 'asc'))
            ->with(['category', 'primaryImage']);
    }
    
    private function clearCache(): void
    {
        Cache::tags(['products:list'])->flush();
    }
    
    private function clearProductCache(int $id, string $slug): void
    {
        Cache::forget('product:' . $id);
        Cache::forget('product:slug:' . $slug);
        Cache::tags(['products:list'])->flush();
    }
}
```

### 2.3 Register Repository ใน Service Container

```php
// app/Providers/RepositoryServiceProvider.php
namespace App\Providers;

use App\Contracts\Repositories\ProductRepositoryInterface;
use App\Repositories\EloquentProductRepository;
use Illuminate\Support\ServiceProvider;

class RepositoryServiceProvider extends ServiceProvider
{
    public array $bindings = [
        ProductRepositoryInterface::class => EloquentProductRepository::class,
    ];
    
    public function register(): void
    {
        foreach ($this->bindings as $abstract => $concrete) {
            $this->app->bind($abstract, $concrete);
        }
    }
}
```

---

## 3. Service Layer

Service Layer เก็บ Business Logic ที่ซับซ้อนไว้รวมกัน

```php
// app/Services/OrderService.php
namespace App\Services;

use App\Contracts\Repositories\OrderRepositoryInterface;
use App\Contracts\Repositories\ProductRepositoryInterface;
use App\DTOs\CreateOrderDTO;
use App\DTOs\OrderItemDTO;
use App\Events\OrderCreated;
use App\Exceptions\InsufficientStockException;
use App\Exceptions\PaymentFailedException;
use App\Models\Order;
use Illuminate\Support\Facades\DB;

class OrderService
{
    public function __construct(
        private readonly OrderRepositoryInterface $orderRepository,
        private readonly ProductRepositoryInterface $productRepository,
        private readonly PaymentService $paymentService,
        private readonly InventoryService $inventoryService,
        private readonly NotificationService $notificationService
    ) {}
    
    public function createOrder(CreateOrderDTO $dto): Order
    {
        // Validate stock availability
        $this->validateStockAvailability($dto->items);
        
        return DB::transaction(function () use ($dto) {
            // Create order
            $order = $this->orderRepository->create([
                'user_id' => $dto->userId,
                'status' => 'pending',
                'subtotal' => $dto->subtotal,
                'tax' => $dto->tax,
                'total' => $dto->total,
                'shipping_address' => $dto->shippingAddress->toArray(),
                'notes' => $dto->notes,
            ]);
            
            // Create order items
            foreach ($dto->items as $item) {
                $order->items()->create([
                    'product_id' => $item->productId,
                    'quantity' => $item->quantity,
                    'price' => $item->price,
                    'subtotal' => $item->subtotal,
                ]);
            }
            
            // Reserve inventory
            foreach ($dto->items as $item) {
                $this->inventoryService->reserve($item->productId, $item->quantity);
            }
            
            // Process payment
            try {
                $payment = $this->paymentService->charge($order, $dto->paymentMethod);
                $order->update(['payment_id' => $payment->id, 'status' => 'processing']);
            } catch (PaymentFailedException $e) {
                // Release reserved inventory
                foreach ($dto->items as $item) {
                    $this->inventoryService->release($item->productId, $item->quantity);
                }
                throw $e;
            }
            
            // Fire event
            event(new OrderCreated($order));
            
            return $order->load(['items.product', 'user']);
        });
    }
    
    public function cancelOrder(Order $order, string $reason): Order
    {
        if (!$order->canBeCancelled()) {
            throw new \RuntimeException("Order #{$order->id} cannot be cancelled");
        }
        
        return DB::transaction(function () use ($order, $reason) {
            // Release inventory
            foreach ($order->items as $item) {
                $this->inventoryService->release($item->product_id, $item->quantity);
            }
            
            // Refund if paid
            if ($order->isPaid()) {
                $this->paymentService->refund($order->payment_id, $order->total);
            }
            
            $order->update([
                'status' => 'cancelled',
                'cancelled_reason' => $reason,
                'cancelled_at' => now(),
            ]);
            
            event(new \App\Events\OrderCancelled($order));
            
            return $order->fresh();
        });
    }
    
    private function validateStockAvailability(array $items): void
    {
        foreach ($items as $item) {
            $product = $this->productRepository->findById($item->productId);
            
            if (!$product) {
                throw new \RuntimeException("Product #{$item->productId} not found");
            }
            
            if ($product->stock < $item->quantity) {
                throw new InsufficientStockException(
                    "Insufficient stock for {$product->name}: available {$product->stock}, requested {$item->quantity}"
                );
            }
        }
    }
}
```

---

## 4. Action Classes

Action Classes ทำตาม Single-Responsibility Principle — หนึ่ง Action ทำสิ่งเดียว

```php
// app/Actions/Orders/CreateOrderAction.php
namespace App\Actions\Orders;

use App\DTOs\CreateOrderData;
use App\Models\Order;
use App\Services\OrderService;

class CreateOrderAction
{
    public function __construct(
        private readonly OrderService $orderService
    ) {}
    
    public function __invoke(CreateOrderData $data): Order
    {
        return $this->orderService->createOrder($data->toDTO());
    }
    
    // หรือใช้ handle() method
    public function handle(CreateOrderData $data): Order
    {
        return $this->orderService->createOrder($data->toDTO());
    }
}
```

```php
// app/Actions/Users/UpdateUserProfileAction.php
namespace App\Actions\Users;

use App\DTOs\UpdateProfileData;
use App\Models\User;
use Illuminate\Support\Facades\Storage;

class UpdateUserProfileAction
{
    public function __invoke(User $user, UpdateProfileData $data): User
    {
        $updateData = [
            'name' => $data->name,
            'bio' => $data->bio,
            'phone' => $data->phone,
        ];
        
        if ($data->photo) {
            // ลบรูปเก่า
            if ($user->photo) {
                Storage::disk('public')->delete($user->photo);
            }
            
            $updateData['photo'] = $data->photo->store('avatars', 'public');
        }
        
        $user->update($updateData);
        
        if ($data->socialLinks) {
            $user->profile()->updateOrCreate(
                ['user_id' => $user->id],
                ['social_links' => $data->socialLinks]
            );
        }
        
        return $user->fresh(['profile']);
    }
}
```

**ใช้งาน Action Classes:**
```php
// app/Http/Controllers/OrderController.php
class OrderController extends Controller
{
    public function __construct(
        private readonly CreateOrderAction $createOrder,
        private readonly CancelOrderAction $cancelOrder
    ) {}
    
    public function store(StoreOrderRequest $request): RedirectResponse
    {
        try {
            $order = ($this->createOrder)(
                CreateOrderData::fromRequest($request)
            );
            
            return redirect()
                ->route('orders.show', $order)
                ->with('success', 'สร้างคำสั่งซื้อเรียบร้อยแล้ว');
        } catch (InsufficientStockException $e) {
            return back()->withErrors(['stock' => $e->getMessage()]);
        } catch (PaymentFailedException $e) {
            return back()->withErrors(['payment' => 'การชำระเงินล้มเหลว: ' . $e->getMessage()]);
        }
    }
}
```

---

## 5. Data Transfer Objects (DTOs)

DTOs ทำให้ Data Flow ชัดเจน Type-safe และ Immutable

### 5.1 สร้าง DTO

```php
// app/DTOs/CreateOrderData.php
namespace App\DTOs;

use Illuminate\Http\Request;
use Spatie\LaravelData\Data;
use Spatie\LaravelData\Attributes\Validation\Rule;
use Spatie\LaravelData\DataCollection;

class CreateOrderData extends Data
{
    public function __construct(
        public readonly int $userId,
        
        /** @var DataCollection<int, OrderItemData> */
        public readonly DataCollection $items,
        
        public readonly ShippingAddressData $shippingAddress,
        
        public readonly PaymentMethodData $paymentMethod,
        
        public readonly ?string $notes = null,
        
        public readonly ?string $couponCode = null,
    ) {}
    
    public static function fromRequest(Request $request): self
    {
        return new self(
            userId: auth()->id(),
            items: OrderItemData::collection($request->items),
            shippingAddress: ShippingAddressData::from($request->shipping_address),
            paymentMethod: PaymentMethodData::from($request->payment_method),
            notes: $request->notes,
            couponCode: $request->coupon_code,
        );
    }
    
    public function getSubtotal(): float
    {
        return $this->items->toCollection()->sum(fn($item) => $item->subtotal);
    }
    
    public function getTax(float $rate = 0.07): float
    {
        return round($this->getSubtotal() * $rate, 2);
    }
    
    public function getTotal(): float
    {
        return $this->getSubtotal() + $this->getTax();
    }
}
```

```php
// app/DTOs/OrderItemData.php
namespace App\DTOs;

use Spatie\LaravelData\Data;
use Spatie\LaravelData\Attributes\Validation\Min;
use Spatie\LaravelData\Attributes\Validation\Exists;

class OrderItemData extends Data
{
    public function __construct(
        #[Exists('products', 'id')]
        public readonly int $productId,
        
        #[Min(1)]
        public readonly int $quantity,
        
        public readonly float $price,
    ) {}
    
    public function getSubtotal(): float
    {
        return $this->price * $this->quantity;
    }
}
```

### 5.2 DTO โดยไม่ใช้ spatie/laravel-data

```php
// app/DTOs/UserProfileData.php
namespace App\DTOs;

use App\Http\Requests\UpdateProfileRequest;
use Illuminate\Http\UploadedFile;

final class UserProfileData
{
    public function __construct(
        public readonly string $name,
        public readonly ?string $bio,
        public readonly ?string $phone,
        public readonly ?UploadedFile $photo = null,
        public readonly ?array $socialLinks = null,
    ) {}
    
    public static function fromRequest(UpdateProfileRequest $request): self
    {
        return new self(
            name: $request->validated('name'),
            bio: $request->validated('bio'),
            phone: $request->validated('phone'),
            photo: $request->file('photo'),
            socialLinks: $request->validated('social_links'),
        );
    }
    
    public function toArray(): array
    {
        return array_filter([
            'name' => $this->name,
            'bio' => $this->bio,
            'phone' => $this->phone,
        ], fn($value) => $value !== null);
    }
}
```

---

## 6. Value Objects

Value Objects คือ Immutable Objects ที่แทนค่า Domain Concept เฉพาะ

```php
// app/ValueObjects/Money.php
namespace App\ValueObjects;

use InvalidArgumentException;

final class Money
{
    public function __construct(
        private readonly int $amount, // เก็บเป็น satang
        private readonly string $currency = 'THB'
    ) {
        if ($amount < 0) {
            throw new InvalidArgumentException('Amount cannot be negative');
        }
    }
    
    public static function fromBaht(float $baht, string $currency = 'THB'): self
    {
        return new self((int) round($baht * 100), $currency);
    }
    
    public function getAmount(): int
    {
        return $this->amount;
    }
    
    public function getBaht(): float
    {
        return $this->amount / 100;
    }
    
    public function getCurrency(): string
    {
        return $this->currency;
    }
    
    public function add(Money $other): self
    {
        $this->assertSameCurrency($other);
        
        return new self($this->amount + $other->amount, $this->currency);
    }
    
    public function subtract(Money $other): self
    {
        $this->assertSameCurrency($other);
        
        if ($other->amount > $this->amount) {
            throw new InvalidArgumentException('Cannot subtract more than available amount');
        }
        
        return new self($this->amount - $other->amount, $this->currency);
    }
    
    public function multiply(float $multiplier): self
    {
        return new self((int) round($this->amount * $multiplier), $this->currency);
    }
    
    public function isGreaterThan(Money $other): bool
    {
        $this->assertSameCurrency($other);
        return $this->amount > $other->amount;
    }
    
    public function equals(Money $other): bool
    {
        return $this->amount === $other->amount && $this->currency === $other->currency;
    }
    
    public function format(): string
    {
        return number_format($this->getBaht(), 2) . ' ' . $this->currency;
    }
    
    public function __toString(): string
    {
        return $this->format();
    }
    
    private function assertSameCurrency(Money $other): void
    {
        if ($this->currency !== $other->currency) {
            throw new InvalidArgumentException(
                "Cannot operate on different currencies: {$this->currency} and {$other->currency}"
            );
        }
    }
}

// Usage
$price = Money::fromBaht(1500.00);
$tax = $price->multiply(0.07);
$total = $price->add($tax);

echo $total->format(); // "1605.00 THB"
```

```php
// app/ValueObjects/Email.php
namespace App\ValueObjects;

final class Email
{
    private readonly string $value;
    
    public function __construct(string $email)
    {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new \InvalidArgumentException("Invalid email: {$email}");
        }
        
        $this->value = strtolower(trim($email));
    }
    
    public function getValue(): string
    {
        return $this->value;
    }
    
    public function getDomain(): string
    {
        return substr($this->value, strpos($this->value, '@') + 1);
    }
    
    public function isGmail(): bool
    {
        return $this->getDomain() === 'gmail.com';
    }
    
    public function equals(Email $other): bool
    {
        return $this->value === $other->value;
    }
    
    public function __toString(): string
    {
        return $this->value;
    }
}
```

---

## 7. Pipeline Pattern

```php
// app/Pipelines/ProcessOrder/Pipeline.php
namespace App\Pipelines\ProcessOrder;

use App\Models\Order;
use Illuminate\Pipeline\Pipeline;
use App\Pipelines\ProcessOrder\Stages\{
    ValidateOrderItems,
    ApplyCoupon,
    CalculateTax,
    CalculateShipping,
    ApplyLoyaltyPoints,
    FinalizeOrder
};

class OrderPipeline
{
    public function __construct(
        private readonly Pipeline $pipeline
    ) {}
    
    public function process(Order $order): Order
    {
        return $this->pipeline
            ->send($order)
            ->through([
                ValidateOrderItems::class,
                ApplyCoupon::class,
                CalculateTax::class,
                CalculateShipping::class,
                ApplyLoyaltyPoints::class,
                FinalizeOrder::class,
            ])
            ->thenReturn();
    }
}
```

```php
// app/Pipelines/ProcessOrder/Stages/ApplyCoupon.php
namespace App\Pipelines\ProcessOrder\Stages;

use App\Models\Order;
use Closure;

class ApplyCoupon
{
    public function handle(Order $order, Closure $next): Order
    {
        if ($order->coupon_code) {
            $coupon = \App\Models\Coupon::where('code', $order->coupon_code)
                ->valid()
                ->first();
            
            if ($coupon) {
                $discount = $coupon->calculateDiscount($order->subtotal);
                
                $order->discount = $discount;
                $order->coupon_id = $coupon->id;
                $order->subtotal -= $discount;
                
                $coupon->increment('used_count');
            }
        }
        
        return $next($order);
    }
}
```

---

## 8. Workshop: Refactor Laravel App ให้ Clean

### Before: Messy Controller

```php
// BEFORE - Fat Controller ทำทุกอย่าง
class UserController extends Controller
{
    public function register(Request $request)
    {
        $request->validate([
            'name' => 'required|string',
            'email' => 'required|email|unique:users',
            'password' => 'required|min:8|confirmed',
            'phone' => 'nullable|string',
        ]);
        
        $user = new User();
        $user->name = $request->name;
        $user->email = $request->email;
        $user->password = bcrypt($request->password);
        $user->phone = $request->phone;
        $user->save();
        
        $profile = new UserProfile();
        $profile->user_id = $user->id;
        $profile->save();
        
        Mail::send('emails.welcome', ['user' => $user], function ($m) use ($user) {
            $m->to($user->email)->subject('Welcome!');
        });
        
        $user->assignRole('customer');
        
        auth()->login($user);
        
        return redirect()->route('dashboard');
    }
}
```

### After: Clean Architecture

```php
// AFTER Step 1: Form Request
// app/Http/Requests/RegisterUserRequest.php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class RegisterUserRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }
    
    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'min:2', 'max:100'],
            'email' => ['required', 'email', 'unique:users,email'],
            'password' => ['required', 'min:8', 'confirmed'],
            'phone' => ['nullable', 'string', 'regex:/^[0-9\-\+\s]{9,15}$/'],
        ];
    }
    
    public function messages(): array
    {
        return [
            'name.required' => 'กรุณากรอกชื่อ',
            'email.unique' => 'อีเมลนี้มีผู้ใช้งานแล้ว',
            'password.min' => 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร',
        ];
    }
}
```

```php
// AFTER Step 2: DTO
// app/DTOs/RegisterUserData.php
namespace App\DTOs;

use App\Http\Requests\RegisterUserRequest;

final class RegisterUserData
{
    public function __construct(
        public readonly string $name,
        public readonly string $email,
        public readonly string $password,
        public readonly ?string $phone = null,
    ) {}
    
    public static function fromRequest(RegisterUserRequest $request): self
    {
        return new self(
            name: $request->validated('name'),
            email: $request->validated('email'),
            password: $request->validated('password'),
            phone: $request->validated('phone'),
        );
    }
}
```

```php
// AFTER Step 3: Action
// app/Actions/Auth/RegisterUserAction.php
namespace App\Actions\Auth;

use App\DTOs\RegisterUserData;
use App\Models\User;
use App\Notifications\WelcomeNotification;
use Illuminate\Support\Facades\DB;

class RegisterUserAction
{
    public function __invoke(RegisterUserData $data): User
    {
        return DB::transaction(function () use ($data) {
            $user = User::create([
                'name' => $data->name,
                'email' => $data->email,
                'password' => bcrypt($data->password),
                'phone' => $data->phone,
            ]);
            
            $user->profile()->create();
            $user->assignRole('customer');
            
            $user->notify(new WelcomeNotification());
            
            return $user;
        });
    }
}
```

```php
// AFTER Step 4: Clean Controller
// app/Http/Controllers/Auth/RegisterController.php
namespace App\Http\Controllers\Auth;

use App\Actions\Auth\RegisterUserAction;
use App\DTOs\RegisterUserData;
use App\Http\Controllers\Controller;
use App\Http\Requests\RegisterUserRequest;
use Illuminate\Http\RedirectResponse;

class RegisterController extends Controller
{
    public function __construct(
        private readonly RegisterUserAction $registerUser
    ) {}
    
    public function store(RegisterUserRequest $request): RedirectResponse
    {
        $user = ($this->registerUser)(
            RegisterUserData::fromRequest($request)
        );
        
        auth()->login($user);
        
        return redirect()
            ->route('dashboard')
            ->with('success', 'ยินดีต้อนรับ ' . $user->name . '!');
    }
}
```

---

## 9. Testing Clean Architecture

```php
// tests/Unit/Actions/RegisterUserActionTest.php
namespace Tests\Unit\Actions;

use App\Actions\Auth\RegisterUserAction;
use App\DTOs\RegisterUserData;
use App\Models\User;
use App\Notifications\WelcomeNotification;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Notification;
use Tests\TestCase;

class RegisterUserActionTest extends TestCase
{
    use RefreshDatabase;
    
    public function test_creates_user_with_correct_data(): void
    {
        $data = new RegisterUserData(
            name: 'John Doe',
            email: 'john@example.com',
            password: 'password123',
            phone: '0812345678',
        );
        
        $action = new RegisterUserAction();
        $user = $action($data);
        
        $this->assertInstanceOf(User::class, $user);
        $this->assertEquals('John Doe', $user->name);
        $this->assertEquals('john@example.com', $user->email);
        $this->assertEquals('0812345678', $user->phone);
    }
    
    public function test_assigns_customer_role(): void
    {
        $data = new RegisterUserData('Test User', 'test@example.com', 'password123');
        
        $action = new RegisterUserAction();
        $user = $action($data);
        
        $this->assertTrue($user->hasRole('customer'));
    }
    
    public function test_sends_welcome_notification(): void
    {
        Notification::fake();
        
        $data = new RegisterUserData('Test User', 'test@example.com', 'password123');
        
        $action = new RegisterUserAction();
        $user = $action($data);
        
        Notification::assertSentTo($user, WelcomeNotification::class);
    }
    
    public function test_creates_user_profile(): void
    {
        $data = new RegisterUserData('Test User', 'test@example.com', 'password123');
        
        $action = new RegisterUserAction();
        $user = $action($data);
        
        $this->assertNotNull($user->profile);
    }
}
```

---

## Quiz

**ข้อ 1:** Repository Pattern ให้ประโยชน์อะไร?

a) ทำให้ Code เร็วขึ้น  
b) Decouple Business Logic จาก Data Access Layer ทำให้เปลี่ยน Database ได้ง่าย  
c) ลด Memory Usage  
d) เพิ่ม Security

**เฉลย:** b) Repository Pattern สร้าง Abstraction Layer ทำให้สามารถเปลี่ยนจาก MySQL เป็น MongoDB หรือ API ได้โดยไม่ต้องแก้ Business Logic

---

**ข้อ 2:** DTO (Data Transfer Object) ต่างจาก Model อย่างไร?

a) DTO ไม่มี Methods  
b) DTO เป็น Immutable Object ที่ใช้ส่งข้อมูลระหว่าง Layer โดยไม่มี Business Logic  
c) DTO เก็บข้อมูลใน Database  
d) DTO เหมือน Model แต่เล็กกว่า

**เฉลย:** b) DTO เป็น Pure Data Container ที่ Immutable ไม่มี Database operations ใช้ส่งข้อมูล Type-safe ระหว่าง Layers

---

**ข้อ 3:** Action Class ควรทำอะไร?

a) ทุกอย่างที่ต้องการ  
b) สิ่งเดียวตาม Single-Responsibility Principle  
c) จัดการ HTTP Requests  
d) เป็น Service Layer

**เฉลย:** b) Action Class ทำสิ่งเดียวตาม Single-Responsibility Principle เช่น CreateUserAction, SendEmailAction

---

## สรุป

ใน Part นี้คุณได้เรียนรู้:
- **Repository Pattern:** Abstraction สำหรับ Data Access
- **Service Layer:** รวม Business Logic ที่ซับซ้อน
- **Action Classes:** Single-Responsibility Operations
- **DTOs:** Type-safe Data Transfer
- **Value Objects:** Immutable Domain Values
- **Workshop:** Refactor Fat Controller เป็น Clean Architecture

---

## ไปต่อ

➡️ [Part 050: Laravel Microservices — แยก Monolith เป็น Services](./part-050-laravel-microservices.md)
