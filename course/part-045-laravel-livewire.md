# Part 045: Laravel Livewire — Reactive UI โดยไม่ต้องเขียน JavaScript

**ระดับ: มืออาชีพ (Professional)**
**เวลาเรียน: 6-8 ชั่วโมง**

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เข้าใจ Livewire Component Lifecycle
- ใช้ Properties, Actions, และ Events
- สร้าง Form Validation แบบ Real-time
- จัดการ File Uploads ด้วย Livewire
- ใช้ Polling และ Lazy Loading
- สร้าง Live Search, Shopping Cart, และ Real-time Dashboard

---

## 1. Livewire คืออะไร?

Laravel Livewire เป็น Full-stack Framework สำหรับสร้าง Dynamic UI โดยเขียน PHP ล้วนๆ ไม่ต้องเขียน JavaScript (ยกเว้นส่วนเล็กน้อย)

**ทำงานอย่างไร:**
1. Component PHP render HTML ฝั่ง Server
2. User ทำ Action → Livewire ส่ง AJAX Request ไป Server
3. Server ประมวลผลและ re-render เฉพาะส่วนที่เปลี่ยน
4. Livewire อัพเดต DOM

```bash
# ติดตั้ง Livewire 3
composer require livewire/livewire

# Livewire 3 ไม่ต้องรัน publish แต่ถ้าต้องการ custom config
php artisan livewire:publish --config
```

---

## 2. Livewire Basics

### 2.1 สร้าง Component แรก

```bash
# สร้าง Component
php artisan make:livewire Counter

# จะสร้างไฟล์ 2 ไฟล์:
# app/Livewire/Counter.php (Component Class)
# resources/views/livewire/counter.blade.php (Template)
```

```php
// app/Livewire/Counter.php
namespace App\Livewire;

use Livewire\Component;

class Counter extends Component
{
    public int $count = 0;
    
    public function increment(): void
    {
        $this->count++;
    }
    
    public function decrement(): void
    {
        $this->count--;
    }
    
    public function reset(): void
    {
        $this->count = 0;
    }
    
    public function render()
    {
        return view('livewire.counter');
    }
}
```

```blade
{{-- resources/views/livewire/counter.blade.php --}}
<div>
    <h2>Count: {{ $count }}</h2>
    
    <button wire:click="increment">+</button>
    <button wire:click="decrement">-</button>
    <button wire:click="reset">Reset</button>
</div>
```

```blade
{{-- resources/views/layouts/app.blade.php --}}
<!DOCTYPE html>
<html>
<head>
    <title>My App</title>
    @livewireStyles
</head>
<body>
    {{-- ใช้งาน Component --}}
    <livewire:counter />
    
    {{-- หรือใช้ Blade Directive --}}
    @livewire('counter')
    
    @livewireScripts
</body>
</html>
```

---

## 3. Component Lifecycle

Livewire มี Lifecycle Hooks หลายตัวที่ทำงานในขั้นตอนต่างๆ

```php
namespace App\Livewire;

use Livewire\Component;
use Livewire\Attributes\On;

class UserProfile extends Component
{
    public int $userId;
    public array $user = [];
    public bool $isLoading = false;
    
    // mount() - เรียกครั้งแรกเมื่อ Component ถูกสร้าง
    public function mount(int $userId): void
    {
        $this->userId = $userId;
        $this->loadUser();
    }
    
    // boot() - เรียกทุกครั้งที่ Component ถูก instantiate
    public function boot(): void
    {
        // เหมาะสำหรับ Dependency Injection
    }
    
    // hydrate() - เรียกหลัง Component ถูก Hydrate จาก JSON (subsequent requests)
    public function hydrate(): void
    {
        // ทำงานหลัง Livewire restore state
    }
    
    // dehydrate() - เรียกก่อน Component ถูก serialize กลับเป็น JSON
    public function dehydrate(): void
    {
        // ทำงานก่อน Livewire serialize state
    }
    
    // updating() - เรียกก่อน property ถูก update
    public function updating(string $name, mixed $value): void
    {
        // เช่น ก่อน $this->search ถูกเปลี่ยน
    }
    
    // updated() - เรียกหลัง property ถูก update
    public function updated(string $name, mixed $value): void
    {
        // เช่น หลัง $this->search ถูกเปลี่ยน
    }
    
    // updatedSearch() - เรียกหลัง $search ถูก update (property-specific)
    public function updatedSearch(string $value): void
    {
        // เฉพาะเจาะจงกว่า updated()
        $this->resetPage();
    }
    
    // rendering() - เรียกก่อน render
    public function rendering(): void {}
    
    // rendered() - เรียกหลัง render
    public function rendered(string $view): void {}
    
    private function loadUser(): void
    {
        $this->user = \App\Models\User::find($this->userId)?->toArray() ?? [];
    }
    
    public function render()
    {
        return view('livewire.user-profile');
    }
}
```

---

## 4. Properties และ Data Binding

### 4.1 Public Properties

```php
namespace App\Livewire;

use Livewire\Component;
use Livewire\Attributes\Locked;
use Livewire\Attributes\Modelable;
use Livewire\Attributes\Url;

class ProductFilter extends Component
{
    // Basic property - synced with template
    public string $search = '';
    
    // Locked - ไม่สามารถ update จาก Frontend ได้
    #[Locked]
    public int $adminId;
    
    // URL - sync กับ Query String
    #[Url]
    public string $sortBy = 'name';
    
    #[Url(as: 'order')]
    public string $sortOrder = 'asc';
    
    // Modelable - ใช้ใน Parent Component
    #[Modelable]
    public string $selectedCategory = '';
    
    public function render()
    {
        return view('livewire.product-filter');
    }
}
```

### 4.2 Data Binding ใน Template

```blade
{{-- resources/views/livewire/product-filter.blade.php --}}
<div>
    {{-- Two-way binding --}}
    <input type="text" wire:model="search" placeholder="Search...">
    
    {{-- Live update (default): update ทุก keystroke --}}
    <input type="text" wire:model.live="search">
    
    {{-- Debounce: รอ 500ms ก่อน update --}}
    <input type="text" wire:model.live.debounce.500ms="search">
    
    {{-- Blur: update เฉพาะเมื่อ blur --}}
    <input type="text" wire:model.blur="search">
    
    {{-- Lazy: update เฉพาะเมื่อ change --}}
    <input type="text" wire:model.lazy="search">
    
    {{-- Select --}}
    <select wire:model="sortBy">
        <option value="name">Name</option>
        <option value="price">Price</option>
        <option value="created_at">Newest</option>
    </select>
    
    {{-- Checkbox --}}
    <input type="checkbox" wire:model="inStock" value="1">
    
    {{-- Radio --}}
    <input type="radio" wire:model="category" value="electronics">
    <input type="radio" wire:model="category" value="clothing">
</div>
```

---

## 5. Actions

### 5.1 Basic Actions

```php
// app/Livewire/TodoList.php
namespace App\Livewire;

use App\Models\Todo;
use Livewire\Component;
use Livewire\Attributes\Validate;

class TodoList extends Component
{
    #[Validate('required|string|min:3|max:255')]
    public string $title = '';
    
    public string $filter = 'all'; // all, active, completed
    
    public function addTodo(): void
    {
        $this->validate();
        
        Todo::create([
            'title' => $this->title,
            'user_id' => auth()->id(),
        ]);
        
        $this->title = '';
        $this->dispatch('todo-added');
    }
    
    public function toggleTodo(int $id): void
    {
        $todo = Todo::findOrFail($id);
        $this->authorize('update', $todo);
        
        $todo->update(['completed' => !$todo->completed]);
    }
    
    public function deleteTodo(int $id): void
    {
        $todo = Todo::findOrFail($id);
        $this->authorize('delete', $todo);
        
        $todo->delete();
    }
    
    public function clearCompleted(): void
    {
        Todo::where('user_id', auth()->id())
            ->where('completed', true)
            ->delete();
    }
    
    public function setFilter(string $filter): void
    {
        $this->filter = $filter;
    }
    
    public function render()
    {
        $todos = Todo::where('user_id', auth()->id())
            ->when($this->filter === 'active', fn($q) => $q->where('completed', false))
            ->when($this->filter === 'completed', fn($q) => $q->where('completed', true))
            ->orderBy('created_at', 'desc')
            ->get();
        
        return view('livewire.todo-list', [
            'todos' => $todos,
            'activeCount' => $todos->where('completed', false)->count(),
        ]);
    }
}
```

```blade
{{-- resources/views/livewire/todo-list.blade.php --}}
<div class="todo-app">
    {{-- Add Todo Form --}}
    <form wire:submit="addTodo">
        <input 
            type="text" 
            wire:model="title"
            placeholder="What needs to be done?"
        >
        @error('title') <span class="error">{{ $message }}</span> @enderror
        <button type="submit">Add</button>
    </form>
    
    {{-- Todo List --}}
    <ul>
        @forelse($todos as $todo)
        <li wire:key="todo-{{ $todo->id }}">
            <input 
                type="checkbox"
                wire:click="toggleTodo({{ $todo->id }})"
                @checked($todo->completed)
            >
            <span class="{{ $todo->completed ? 'line-through' : '' }}">
                {{ $todo->title }}
            </span>
            <button wire:click="deleteTodo({{ $todo->id }})" wire:confirm="Delete this todo?">
                Delete
            </button>
        </li>
        @empty
        <li>No todos yet!</li>
        @endforelse
    </ul>
    
    {{-- Footer --}}
    <div class="footer">
        <span>{{ $activeCount }} items left</span>
        
        <div class="filters">
            <button 
                wire:click="setFilter('all')"
                class="{{ $filter === 'all' ? 'active' : '' }}"
            >All</button>
            <button 
                wire:click="setFilter('active')"
                class="{{ $filter === 'active' ? 'active' : '' }}"
            >Active</button>
            <button 
                wire:click="setFilter('completed')"
                class="{{ $filter === 'completed' ? 'active' : '' }}"
            >Completed</button>
        </div>
        
        <button wire:click="clearCompleted">Clear completed</button>
    </div>
    
    {{-- Loading indicator --}}
    <div wire:loading.class="visible" class="loading-overlay">
        Loading...
    </div>
</div>
```

### 5.2 Confirmation Dialogs

```blade
{{-- wire:confirm - ยืนยันก่อนทำงาน --}}
<button 
    wire:click="deleteAccount"
    wire:confirm="Are you sure? This cannot be undone!"
>
    Delete Account
</button>

{{-- Custom confirm dialog --}}
<button 
    wire:click="$dispatch('confirm-delete', { id: {{ $item->id }} })"
>
    Delete
</button>
```

---

## 6. Form Validation

### 6.1 Validation Rules

```php
namespace App\Livewire;

use Livewire\Component;
use Livewire\Attributes\Validate;
use Livewire\Attributes\Rule;

class RegistrationForm extends Component
{
    // Attribute-based validation
    #[Validate('required|string|min:2|max:50')]
    public string $name = '';
    
    #[Validate('required|email|unique:users,email')]
    public string $email = '';
    
    #[Validate('required|string|min:8|confirmed')]
    public string $password = '';
    
    #[Validate('required|string|min:8')]
    public string $password_confirmation = '';
    
    // Computed rules
    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'min:2', 'max:50'],
            'email' => ['required', 'email', Rule::unique('users')->ignore($this->userId)],
            'password' => ['required', 'min:8', 'confirmed'],
        ];
    }
    
    // Custom messages
    public function messages(): array
    {
        return [
            'name.required' => 'กรุณากรอกชื่อ',
            'email.required' => 'กรุณากรอกอีเมล',
            'email.email' => 'รูปแบบอีเมลไม่ถูกต้อง',
            'email.unique' => 'อีเมลนี้มีผู้ใช้งานแล้ว',
            'password.required' => 'กรุณากรอกรหัสผ่าน',
            'password.min' => 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร',
        ];
    }
    
    // Validation attribute names
    public function validationAttributes(): array
    {
        return [
            'name' => 'ชื่อ',
            'email' => 'อีเมล',
            'password' => 'รหัสผ่าน',
        ];
    }
    
    public function register(): void
    {
        $validated = $this->validate();
        
        $user = \App\Models\User::create([
            'name' => $validated['name'],
            'email' => $validated['email'],
            'password' => bcrypt($validated['password']),
        ]);
        
        auth()->login($user);
        
        $this->redirect(route('dashboard'), navigate: true);
    }
    
    // Real-time validation per property
    public function updatedEmail(): void
    {
        $this->validateOnly('email');
    }
    
    public function render()
    {
        return view('livewire.registration-form');
    }
}
```

```blade
{{-- resources/views/livewire/registration-form.blade.php --}}
<div>
    <form wire:submit="register">
        <div>
            <label>ชื่อ</label>
            <input 
                type="text" 
                wire:model.live.debounce.300ms="name"
                class="{{ $errors->has('name') ? 'border-red-500' : '' }}"
            >
            @error('name')
                <p class="text-red-500 text-sm">{{ $message }}</p>
            @enderror
        </div>
        
        <div>
            <label>อีเมล</label>
            <input 
                type="email" 
                wire:model.blur="email"
                class="{{ $errors->has('email') ? 'border-red-500' : '' }}"
            >
            @error('email')
                <p class="text-red-500 text-sm">{{ $message }}</p>
            @enderror
        </div>
        
        <div>
            <label>รหัสผ่าน</label>
            <input 
                type="password" 
                wire:model.blur="password"
            >
            @error('password')
                <p class="text-red-500 text-sm">{{ $message }}</p>
            @enderror
        </div>
        
        <div>
            <label>ยืนยันรหัสผ่าน</label>
            <input 
                type="password" 
                wire:model.blur="password_confirmation"
            >
        </div>
        
        <button 
            type="submit"
            wire:loading.attr="disabled"
        >
            <span wire:loading.remove>สมัครสมาชิก</span>
            <span wire:loading>กำลังดำเนินการ...</span>
        </button>
    </form>
</div>
```

---

## 7. File Uploads

### 7.1 Basic File Upload

```php
namespace App\Livewire;

use Livewire\Component;
use Livewire\WithFileUploads;
use Livewire\Attributes\Validate;

class ProfilePhoto extends Component
{
    use WithFileUploads;
    
    #[Validate('image|max:2048')] // 2MB max
    public $photo;
    
    public function save(): void
    {
        $this->validate();
        
        $path = $this->photo->store('photos', 'public');
        
        auth()->user()->update(['photo' => $path]);
        
        $this->reset('photo');
        
        session()->flash('message', 'อัพโหลดรูปภาพสำเร็จ');
    }
    
    public function render()
    {
        return view('livewire.profile-photo');
    }
}
```

### 7.2 Multiple File Upload

```php
namespace App\Livewire;

use Livewire\Component;
use Livewire\WithFileUploads;
use Livewire\Attributes\Validate;

class ProductImages extends Component
{
    use WithFileUploads;
    
    #[Validate(['photos.*' => 'image|max:5120'])] // 5MB each
    public array $photos = [];
    
    public array $uploadedUrls = [];
    
    public function uploadPhotos(): void
    {
        $this->validate();
        
        foreach ($this->photos as $photo) {
            $path = $photo->store('products', 'public');
            $this->uploadedUrls[] = Storage::url($path);
        }
        
        $this->reset('photos');
        
        $this->dispatch('photos-uploaded', urls: $this->uploadedUrls);
    }
    
    public function removePhoto(int $index): void
    {
        array_splice($this->uploadedUrls, $index, 1);
    }
    
    public function render()
    {
        return view('livewire.product-images');
    }
}
```

```blade
{{-- resources/views/livewire/product-images.blade.php --}}
<div>
    <form wire:submit="uploadPhotos">
        <input 
            type="file" 
            wire:model="photos" 
            multiple
            accept="image/*"
        >
        
        {{-- Preview ภาพก่อน Upload --}}
        <div class="grid grid-cols-4 gap-2 mt-4">
            @foreach($photos as $photo)
            <div>
                <img src="{{ $photo->temporaryUrl() }}" class="w-full h-32 object-cover">
            </div>
            @endforeach
        </div>
        
        {{-- Upload Progress --}}
        <div wire:loading wire:target="photos">
            <div class="progress-bar">
                กำลังอัพโหลด...
            </div>
        </div>
        
        <button type="submit" wire:loading.attr="disabled" wire:target="uploadPhotos">
            อัพโหลดรูปภาพ
        </button>
    </form>
    
    {{-- Uploaded Images --}}
    @if($uploadedUrls)
    <div class="mt-4">
        <h3>อัพโหลดแล้ว:</h3>
        <div class="grid grid-cols-4 gap-2">
            @foreach($uploadedUrls as $index => $url)
            <div class="relative">
                <img src="{{ $url }}" class="w-full h-32 object-cover">
                <button 
                    wire:click="removePhoto({{ $index }})"
                    class="absolute top-1 right-1 bg-red-500 text-white rounded-full w-6 h-6"
                >×</button>
            </div>
            @endforeach
        </div>
    </div>
    @endif
</div>
```

---

## 8. Polling และ Lazy Loading

### 8.1 Polling

```blade
{{-- Poll ทุก 5 วินาที --}}
<div wire:poll.5s="refreshNotifications">
    @foreach($notifications as $notification)
        <div>{{ $notification->message }}</div>
    @endforeach
</div>

{{-- Poll เฉพาะ method --}}
<div wire:poll.10s="checkForUpdates">
    Last updated: {{ $lastUpdated }}
</div>
```

```php
namespace App\Livewire;

use Livewire\Component;

class NotificationBell extends Component
{
    public int $unreadCount = 0;
    public array $notifications = [];
    
    public function mount(): void
    {
        $this->refreshNotifications();
    }
    
    public function refreshNotifications(): void
    {
        $this->unreadCount = auth()->user()
            ->notifications()
            ->whereNull('read_at')
            ->count();
        
        $this->notifications = auth()->user()
            ->notifications()
            ->latest()
            ->take(5)
            ->get()
            ->toArray();
    }
    
    public function markAllAsRead(): void
    {
        auth()->user()->notifications()->update(['read_at' => now()]);
        $this->refreshNotifications();
    }
    
    public function render()
    {
        return view('livewire.notification-bell');
    }
}
```

### 8.2 Lazy Loading

```php
// app/Livewire/ExpensiveChart.php
namespace App\Livewire;

use Livewire\Component;
use Livewire\Attributes\Lazy;

#[Lazy]
class ExpensiveChart extends Component
{
    public function render()
    {
        // คำนวณข้อมูล Charts ที่ใช้เวลานาน
        $data = $this->calculateChartData();
        
        return view('livewire.expensive-chart', compact('data'));
    }
    
    private function calculateChartData(): array
    {
        return \App\Models\Order::selectRaw('DATE(created_at) as date, SUM(total) as revenue')
            ->where('created_at', '>=', now()->subDays(30))
            ->groupBy('date')
            ->get()
            ->toArray();
    }
    
    // Placeholder ระหว่างโหลด
    public function placeholder()
    {
        return <<<'HTML'
        <div class="animate-pulse bg-gray-200 h-64 rounded"></div>
        HTML;
    }
}
```

---

## 9. Events และ Listeners

### 9.1 Component Events

```php
// Parent Component
namespace App\Livewire;

use Livewire\Component;
use Livewire\Attributes\On;

class ShoppingCart extends Component
{
    public array $items = [];
    
    // รับ event จาก Child Components
    #[On('product-added')]
    public function addProduct(int $productId, int $quantity = 1): void
    {
        $product = \App\Models\Product::find($productId);
        
        if (!$product) return;
        
        $existingKey = collect($this->items)->search(fn($item) => $item['id'] === $productId);
        
        if ($existingKey !== false) {
            $this->items[$existingKey]['quantity'] += $quantity;
        } else {
            $this->items[] = [
                'id' => $product->id,
                'name' => $product->name,
                'price' => $product->price,
                'quantity' => $quantity,
            ];
        }
    }
    
    #[On('cart-cleared')]
    public function clearCart(): void
    {
        $this->items = [];
    }
    
    public function removeItem(int $index): void
    {
        array_splice($this->items, $index, 1);
    }
    
    public function getTotal(): float
    {
        return collect($this->items)->sum(fn($item) => $item['price'] * $item['quantity']);
    }
    
    public function render()
    {
        return view('livewire.shopping-cart', [
            'total' => $this->getTotal(),
        ]);
    }
}
```

```php
// Child Component - Product Card
namespace App\Livewire;

use Livewire\Component;

class ProductCard extends Component
{
    public array $product;
    public int $quantity = 1;
    
    public function addToCart(): void
    {
        // ส่ง event ไป Parent/Siblings
        $this->dispatch('product-added', productId: $this->product['id'], quantity: $this->quantity);
        
        // Flash message
        session()->flash('cart-message', 'เพิ่มสินค้าลงตะกร้าแล้ว!');
    }
    
    public function render()
    {
        return view('livewire.product-card');
    }
}
```

### 9.2 JavaScript Events

```blade
{{-- ส่ง event ไป JavaScript --}}
<button wire:click="$dispatch('show-modal', { id: {{ $item->id }} })">
    Open Modal
</button>
```

```javascript
// รับ event จาก Livewire ใน JavaScript
document.addEventListener('show-modal', (event) => {
    const modal = document.getElementById('modal-' + event.detail.id);
    modal.classList.remove('hidden');
});

// ส่ง event จาก JavaScript ไป Livewire
Livewire.dispatch('refresh-data');
```

---

## 10. Pagination

```php
namespace App\Livewire;

use App\Models\Product;
use Livewire\Component;
use Livewire\WithPagination;
use Livewire\Attributes\Url;

class ProductList extends Component
{
    use WithPagination;
    
    #[Url]
    public string $search = '';
    
    #[Url]
    public string $category = '';
    
    #[Url]
    public string $sortBy = 'created_at';
    
    // Reset page เมื่อ search หรือ filter เปลี่ยน
    public function updatedSearch(): void
    {
        $this->resetPage();
    }
    
    public function updatedCategory(): void
    {
        $this->resetPage();
    }
    
    public function render()
    {
        $products = Product::query()
            ->when($this->search, fn($q) => $q->where('name', 'like', '%' . $this->search . '%'))
            ->when($this->category, fn($q) => $q->where('category_id', $this->category))
            ->orderBy($this->sortBy, 'desc')
            ->paginate(12);
        
        $categories = \App\Models\Category::all();
        
        return view('livewire.product-list', compact('products', 'categories'));
    }
}
```

```blade
{{-- resources/views/livewire/product-list.blade.php --}}
<div>
    {{-- Filters --}}
    <div class="filters">
        <input 
            type="text"
            wire:model.live.debounce.300ms="search"
            placeholder="ค้นหาสินค้า..."
        >
        
        <select wire:model.live="category">
            <option value="">ทุกหมวดหมู่</option>
            @foreach($categories as $cat)
            <option value="{{ $cat->id }}">{{ $cat->name }}</option>
            @endforeach
        </select>
    </div>
    
    {{-- Products Grid --}}
    <div class="grid grid-cols-3 gap-4">
        @forelse($products as $product)
        <div wire:key="product-{{ $product->id }}">
            <livewire:product-card :product="$product->toArray()" :key="$product->id" />
        </div>
        @empty
        <div class="col-span-3 text-center py-8">
            ไม่พบสินค้า
        </div>
        @endforelse
    </div>
    
    {{-- Pagination --}}
    {{ $products->links() }}
</div>
```

---

## 11. Workshop 1: Live Search

```php
// app/Livewire/LiveSearch.php
namespace App\Livewire;

use App\Models\Product;
use Livewire\Component;

class LiveSearch extends Component
{
    public string $query = '';
    public bool $showResults = false;
    
    public function getResultsProperty()
    {
        if (strlen($this->query) < 2) {
            return collect();
        }
        
        return Product::where('name', 'like', '%' . $this->query . '%')
            ->orWhere('sku', 'like', '%' . $this->query . '%')
            ->with('category')
            ->take(10)
            ->get();
    }
    
    public function updatedQuery(): void
    {
        $this->showResults = strlen($this->query) >= 2;
    }
    
    public function selectProduct(int $id): void
    {
        $this->redirect(route('products.show', $id));
    }
    
    public function closeResults(): void
    {
        $this->showResults = false;
    }
    
    public function render()
    {
        return view('livewire.live-search');
    }
}
```

```blade
{{-- resources/views/livewire/live-search.blade.php --}}
<div class="relative" x-data @click.outside="$wire.closeResults()">
    <input 
        type="text"
        wire:model.live.debounce.300ms="query"
        wire:focus="$set('showResults', true)"
        placeholder="ค้นหาสินค้า..."
        class="w-full px-4 py-2 border rounded-lg"
    >
    
    {{-- Loading --}}
    <div wire:loading wire:target="query" class="absolute right-3 top-3">
        <svg class="animate-spin h-5 w-5 text-gray-400" fill="none" viewBox="0 0 24 24">
            <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
            <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z"></path>
        </svg>
    </div>
    
    {{-- Results Dropdown --}}
    @if($showResults && $this->results->count() > 0)
    <div class="absolute w-full bg-white border rounded-lg shadow-lg z-50 mt-1">
        @foreach($this->results as $product)
        <div 
            wire:click="selectProduct({{ $product->id }})"
            class="flex items-center p-3 hover:bg-gray-50 cursor-pointer"
        >
            @if($product->image)
            <img src="{{ $product->image_url }}" class="w-12 h-12 object-cover rounded mr-3">
            @endif
            <div>
                <div class="font-medium">{{ $product->name }}</div>
                <div class="text-sm text-gray-500">{{ $product->category->name }}</div>
            </div>
            <div class="ml-auto font-bold">฿{{ number_format($product->price) }}</div>
        </div>
        @endforeach
    </div>
    @elseif($showResults && strlen($query) >= 2)
    <div class="absolute w-full bg-white border rounded-lg shadow-lg z-50 mt-1 p-3 text-gray-500">
        ไม่พบสินค้าที่ตรงกัน
    </div>
    @endif
</div>
```

---

## 12. Workshop 2: Shopping Cart

```php
// app/Livewire/Cart.php
namespace App\Livewire;

use App\Models\Product;
use Livewire\Component;
use Livewire\Attributes\On;
use Illuminate\Support\Collection;

class Cart extends Component
{
    public array $items = [];
    
    public function mount(): void
    {
        $this->items = session()->get('cart', []);
    }
    
    #[On('add-to-cart')]
    public function addItem(int $productId, int $quantity = 1): void
    {
        $product = Product::find($productId);
        
        if (!$product || !$product->in_stock) {
            $this->dispatch('cart-error', message: 'สินค้าหมด');
            return;
        }
        
        $existingIndex = collect($this->items)
            ->search(fn($item) => $item['product_id'] === $productId);
        
        if ($existingIndex !== false) {
            $newQuantity = $this->items[$existingIndex]['quantity'] + $quantity;
            
            if ($newQuantity > $product->stock) {
                $this->dispatch('cart-error', message: 'สินค้าในสต็อกมีไม่เพียงพอ');
                return;
            }
            
            $this->items[$existingIndex]['quantity'] = $newQuantity;
        } else {
            $this->items[] = [
                'product_id' => $product->id,
                'name' => $product->name,
                'price' => $product->price,
                'image' => $product->primary_image_url,
                'quantity' => $quantity,
                'max_quantity' => $product->stock,
            ];
        }
        
        $this->saveToSession();
        $this->dispatch('cart-updated', count: $this->getItemCount());
    }
    
    public function updateQuantity(int $index, int $quantity): void
    {
        if ($quantity <= 0) {
            $this->removeItem($index);
            return;
        }
        
        if ($quantity > $this->items[$index]['max_quantity']) {
            $this->dispatch('cart-error', message: 'จำนวนเกินสต็อก');
            return;
        }
        
        $this->items[$index]['quantity'] = $quantity;
        $this->saveToSession();
    }
    
    public function removeItem(int $index): void
    {
        array_splice($this->items, $index, 1);
        $this->saveToSession();
        $this->dispatch('cart-updated', count: $this->getItemCount());
    }
    
    public function clearCart(): void
    {
        $this->items = [];
        session()->forget('cart');
        $this->dispatch('cart-updated', count: 0);
    }
    
    public function checkout(): void
    {
        if (empty($this->items)) {
            $this->dispatch('cart-error', message: 'ตะกร้าสินค้าว่าง');
            return;
        }
        
        $this->redirect(route('checkout'));
    }
    
    private function saveToSession(): void
    {
        session()->put('cart', $this->items);
    }
    
    public function getItemCount(): int
    {
        return collect($this->items)->sum('quantity');
    }
    
    public function getSubtotal(): float
    {
        return collect($this->items)->sum(fn($item) => $item['price'] * $item['quantity']);
    }
    
    public function getTax(): float
    {
        return $this->getSubtotal() * 0.07;
    }
    
    public function getTotal(): float
    {
        return $this->getSubtotal() + $this->getTax();
    }
    
    public function render()
    {
        return view('livewire.cart');
    }
}
```

---

## 13. Workshop 3: Real-time Dashboard

```php
// app/Livewire/Dashboard.php
namespace App\Livewire;

use App\Models\Order;
use App\Models\User;
use App\Models\Product;
use Livewire\Component;

class Dashboard extends Component
{
    public array $stats = [];
    public array $recentOrders = [];
    public array $topProducts = [];
    public string $period = 'today';
    
    public function mount(): void
    {
        $this->loadData();
    }
    
    public function loadData(): void
    {
        $this->stats = $this->getStats();
        $this->recentOrders = $this->getRecentOrders();
        $this->topProducts = $this->getTopProducts();
    }
    
    public function setPeriod(string $period): void
    {
        $this->period = $period;
        $this->loadData();
    }
    
    private function getStats(): array
    {
        $dateFilter = match($this->period) {
            'today' => now()->startOfDay(),
            'week' => now()->startOfWeek(),
            'month' => now()->startOfMonth(),
            'year' => now()->startOfYear(),
            default => now()->startOfDay(),
        };
        
        return [
            'total_revenue' => Order::where('created_at', '>=', $dateFilter)
                ->where('status', 'completed')
                ->sum('total'),
            'total_orders' => Order::where('created_at', '>=', $dateFilter)->count(),
            'new_customers' => User::where('created_at', '>=', $dateFilter)->count(),
            'pending_orders' => Order::where('status', 'pending')->count(),
            'revenue_change' => $this->calculateRevenueChange($dateFilter),
        ];
    }
    
    private function getRecentOrders(): array
    {
        return Order::with(['user', 'items'])
            ->latest()
            ->take(10)
            ->get()
            ->map(fn($order) => [
                'id' => $order->id,
                'customer' => $order->user->name,
                'total' => $order->total,
                'status' => $order->status,
                'created_at' => $order->created_at->diffForHumans(),
            ])
            ->toArray();
    }
    
    private function getTopProducts(): array
    {
        return Product::withCount(['orderItems as sold_count'])
            ->withSum(['orderItems as revenue' => function ($q) {
                $q->join('orders', 'order_items.order_id', '=', 'orders.id')
                  ->where('orders.status', 'completed');
            }], 'subtotal')
            ->orderByDesc('sold_count')
            ->take(5)
            ->get()
            ->toArray();
    }
    
    private function calculateRevenueChange(mixed $since): float
    {
        $currentRevenue = Order::where('created_at', '>=', $since)
            ->where('status', 'completed')
            ->sum('total');
        
        $previousPeriod = clone $since;
        $diff = now()->diffInSeconds($since);
        $previousStart = $since->copy()->subSeconds($diff);
        
        $previousRevenue = Order::whereBetween('created_at', [$previousStart, $since])
            ->where('status', 'completed')
            ->sum('total');
        
        if ($previousRevenue == 0) return 0;
        
        return round(($currentRevenue - $previousRevenue) / $previousRevenue * 100, 2);
    }
    
    public function render()
    {
        return view('livewire.dashboard');
    }
}
```

```blade
{{-- resources/views/livewire/dashboard.blade.php --}}
<div wire:poll.30s="loadData">
    {{-- Period Selector --}}
    <div class="flex gap-2 mb-6">
        @foreach(['today' => 'วันนี้', 'week' => 'สัปดาห์นี้', 'month' => 'เดือนนี้', 'year' => 'ปีนี้'] as $value => $label)
        <button
            wire:click="setPeriod('{{ $value }}')"
            class="{{ $period === $value ? 'bg-blue-500 text-white' : 'bg-white text-gray-700' }} px-4 py-2 rounded border"
        >
            {{ $label }}
        </button>
        @endforeach
    </div>
    
    {{-- Stats Cards --}}
    <div class="grid grid-cols-4 gap-4 mb-6">
        <div class="bg-white rounded-lg p-4 shadow">
            <div class="text-sm text-gray-500">รายได้</div>
            <div class="text-2xl font-bold">฿{{ number_format($stats['total_revenue']) }}</div>
            <div class="text-sm {{ $stats['revenue_change'] >= 0 ? 'text-green-500' : 'text-red-500' }}">
                {{ $stats['revenue_change'] >= 0 ? '+' : '' }}{{ $stats['revenue_change'] }}%
            </div>
        </div>
        
        <div class="bg-white rounded-lg p-4 shadow">
            <div class="text-sm text-gray-500">คำสั่งซื้อ</div>
            <div class="text-2xl font-bold">{{ number_format($stats['total_orders']) }}</div>
        </div>
        
        <div class="bg-white rounded-lg p-4 shadow">
            <div class="text-sm text-gray-500">ลูกค้าใหม่</div>
            <div class="text-2xl font-bold">{{ number_format($stats['new_customers']) }}</div>
        </div>
        
        <div class="bg-white rounded-lg p-4 shadow">
            <div class="text-sm text-gray-500">รอดำเนินการ</div>
            <div class="text-2xl font-bold text-yellow-500">{{ number_format($stats['pending_orders']) }}</div>
        </div>
    </div>
    
    {{-- Recent Orders --}}
    <div class="bg-white rounded-lg shadow p-4">
        <h3 class="font-bold mb-4">คำสั่งซื้อล่าสุด</h3>
        <table class="w-full">
            <thead>
                <tr class="text-left text-gray-500 text-sm">
                    <th>Order #</th>
                    <th>ลูกค้า</th>
                    <th>ยอด</th>
                    <th>สถานะ</th>
                    <th>เวลา</th>
                </tr>
            </thead>
            <tbody>
                @foreach($recentOrders as $order)
                <tr class="border-t">
                    <td>#{{ $order['id'] }}</td>
                    <td>{{ $order['customer'] }}</td>
                    <td>฿{{ number_format($order['total']) }}</td>
                    <td>
                        <span class="px-2 py-1 rounded text-xs
                            {{ $order['status'] === 'completed' ? 'bg-green-100 text-green-800' : 
                               ($order['status'] === 'pending' ? 'bg-yellow-100 text-yellow-800' : 'bg-gray-100') }}">
                            {{ $order['status'] }}
                        </span>
                    </td>
                    <td class="text-gray-500 text-sm">{{ $order['created_at'] }}</td>
                </tr>
                @endforeach
            </tbody>
        </table>
    </div>
    
    {{-- Loading Overlay --}}
    <div wire:loading class="fixed inset-0 bg-black bg-opacity-20 flex items-center justify-center z-50">
        <div class="bg-white rounded-lg p-4 flex items-center">
            <svg class="animate-spin h-5 w-5 text-blue-500 mr-2" fill="none" viewBox="0 0 24 24">
                <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z"></path>
            </svg>
            กำลังโหลดข้อมูล...
        </div>
    </div>
</div>
```

---

## Quiz

**ข้อ 1:** `wire:model.live` ต่างจาก `wire:model` อย่างไร?

a) `wire:model.live` อัพเดตทุก keystroke, `wire:model` อัพเดตเมื่อ submit  
b) `wire:model.live` ใช้ WebSocket, `wire:model` ใช้ AJAX  
c) ไม่มีความแตกต่าง  
d) `wire:model` เร็วกว่า

**เฉลย:** a) `wire:model.live` ส่ง request ทุกครั้งที่ input เปลี่ยน ส่วน `wire:model` (lazy) ส่งเมื่อ form submit หรือ blur

---

**ข้อ 2:** Livewire Lifecycle Hook ใดที่เรียกทุกครั้งที่ Component ถูก instantiate?

a) `mount()`  
b) `boot()`  
c) `hydrate()`  
d) `dehydrate()`

**เฉลย:** b) `boot()` เรียกทุก request, `mount()` เรียกเฉพาะ initial render

---

**ข้อ 3:** เมื่อใช้ `WithPagination` ควรทำอะไรเมื่อ search parameter เปลี่ยน?

a) ไม่ต้องทำอะไร Pagination จัดการเอง  
b) เรียก `resetPage()`  
c) เรียก `clearPage()`  
d) เรียก `$this->page = 1`

**เฉลย:** b) เรียก `resetPage()` ใน `updatedSearch()` เพื่อกลับไป page 1 เมื่อ search เปลี่ยน

---

## สรุป

ใน Part นี้คุณได้เรียนรู้:
- **Livewire Basics:** Component, Properties, Actions
- **Lifecycle:** mount, boot, hydrate, updated hooks
- **Form Validation:** Real-time validation ด้วย Attribute-based rules
- **File Uploads:** Single และ Multiple file uploads
- **Workshop:** Live Search, Shopping Cart, Real-time Dashboard

---

## ไปต่อ

➡️ [Part 046: Laravel Package Development — สร้าง Package ของตัวเอง](./part-046-laravel-packages.md)
