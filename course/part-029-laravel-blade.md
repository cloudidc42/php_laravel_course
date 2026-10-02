# Part 029: Laravel Blade Templates

**ระดับ: กลาง (Intermediate)**

---

## เป้าหมายของ Part นี้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- ใช้ Blade syntax พื้นฐานได้อย่างถูกต้อง
- สร้าง Template Inheritance ด้วย layouts
- สร้างและใช้ Blade Components
- ใช้ loops, conditionals และ directives ใน Blade
- สร้าง Custom Directives ได้เอง
- สร้าง Layout สมบูรณ์สำหรับ Blog

---

## 1. Blade Syntax พื้นฐาน

### แสดงข้อมูล (Echoing)

```html
<!-- resources/views/example.blade.php -->

{{-- ความคิดเห็น (Blade comment - ไม่แสดงใน HTML) --}}

<!-- แสดงค่าตัวแปร (XSS-safe, escape HTML) -->
<p>{{ $name }}</p>
<p>{{ $post->title }}</p>
<p>{{ $user->profile->bio ?? 'ยังไม่ได้กรอกข้อมูล' }}</p>

<!-- Ternary operator -->
<p>{{ $user->isAdmin() ? 'ผู้ดูแลระบบ' : 'สมาชิกทั่วไป' }}</p>

<!-- แสดง HTML โดยไม่ escape (ระวัง XSS!) -->
{!! $post->content !!}

<!-- Default value -->
{{ $variable ?? 'ค่าเริ่มต้น' }}

<!-- Escaped Blade syntax (แสดง {{ }} จริงๆ ไม่ render) -->
@{{ this is not rendered by Blade }}
```

### PHP ใน Blade

```html
<!-- รัน PHP code -->
@php
    $today = now()->format('d/m/Y');
    $greeting = now()->hour < 12 ? 'สวัสดีตอนเช้า' : 'สวัสดีตอนบ่าย';
@endphp

<p>{{ $greeting }}, วันนี้คือ {{ $today }}</p>

<!-- หรือใช้ inline PHP -->
<?php $count = count($items); ?>
<p>จำนวน: {{ $count }}</p>
```

---

## 2. Conditionals ใน Blade

### @if, @elseif, @else

```html
@if ($user->isAdmin())
    <span class="badge badge-admin">ผู้ดูแลระบบ</span>
@elseif ($user->isModerator())
    <span class="badge badge-mod">ผู้ดูแล</span>
@else
    <span class="badge badge-user">สมาชิก</span>
@endif

<!-- @unless (ตรงข้ามกับ @if) -->
@unless ($user->isVerified())
    <div class="alert alert-warning">
        กรุณา verify email ของคุณ
        <a href="{{ route('verification.notice') }}">คลิกที่นี่</a>
    </div>
@endunless

<!-- @isset - ตรวจสอบว่าตัวแปรถูก set และไม่ใช่ null -->
@isset($post->image)
    <img src="{{ asset('storage/' . $post->image) }}" alt="{{ $post->title }}">
@endisset

<!-- @empty - ตรวจสอบว่า empty -->
@empty($posts)
    <p>ยังไม่มีบทความ</p>
@endempty

<!-- @env - ตรวจสอบ environment -->
@env('local')
    <div class="debug-bar">Debug mode</div>
@endenv
```

### @auth และ @guest

```html
<!-- แสดงเมื่อ login แล้ว -->
@auth
    <a href="{{ route('profile') }}">{{ auth()->user()->name }}</a>
    <form method="POST" action="{{ route('logout') }}">
        @csrf
        <button type="submit">ออกจากระบบ</button>
    </form>
@endauth

<!-- แสดงเมื่อยังไม่ login -->
@guest
    <a href="{{ route('login') }}">เข้าสู่ระบบ</a>
    <a href="{{ route('register') }}">สมัครสมาชิก</a>
@endguest

<!-- ระบุ guard -->
@auth('admin')
    <p>Admin Dashboard</p>
@endauth
```

### @can และ @cannot (Authorization)

```html
<!-- ตรวจสอบ Policy -->
@can('update', $post)
    <a href="{{ route('posts.edit', $post) }}" class="btn btn-edit">แก้ไข</a>
@endcan

@can('delete', $post)
    <form method="POST" action="{{ route('posts.destroy', $post) }}">
        @csrf
        @method('DELETE')
        <button type="submit" class="btn btn-delete" 
                onclick="return confirm('ต้องการลบบทความนี้?')">
            ลบ
        </button>
    </form>
@endcan

@cannot('update', $post)
    <p class="text-muted">คุณไม่มีสิทธิ์แก้ไขบทความนี้</p>
@endcannot

<!-- @canany - ตรวจสอบหลาย permissions -->
@canany(['update', 'delete'], $post)
    <div class="post-actions">...</div>
@endcanany
```

---

## 3. Loops ใน Blade

### @foreach

```html
<!-- @foreach พื้นฐาน -->
@foreach ($posts as $post)
    <article class="post">
        <h2>{{ $post->title }}</h2>
        <p>{{ $post->excerpt }}</p>
    </article>
@endforeach

<!-- @foreach พร้อม @empty -->
@forelse ($posts as $post)
    <article>
        <h2>{{ $post->title }}</h2>
    </article>
@empty
    <div class="empty-state">
        <p>ยังไม่มีบทความ</p>
        @auth
            <a href="{{ route('posts.create') }}">สร้างบทความแรกของคุณ</a>
        @endauth
    </div>
@endforelse
```

### $loop Variable

```html
@foreach ($posts as $post)
    <div class="post 
        {{ $loop->first ? 'first-post' : '' }} 
        {{ $loop->last ? 'last-post' : '' }}
        {{ $loop->even ? 'even' : 'odd' }}">
        
        <!-- ลำดับ (1-based) -->
        <span class="number">{{ $loop->iteration }}</span>
        
        <!-- index (0-based) -->
        <span class="index">{{ $loop->index }}</span>
        
        <!-- จำนวนที่เหลือ -->
        <span>เหลือ {{ $loop->remaining }} รายการ</span>
        
        <!-- จำนวนทั้งหมด -->
        <span>จากทั้งหมด {{ $loop->count }}</span>
        
        <h2>{{ $post->title }}</h2>
        
        @if ($loop->iteration % 3 === 0)
            <div class="ad-banner">โฆษณา</div>
        @endif
    </div>
@endforeach
```

### Nested Loops

```html
@foreach ($categories as $category)
    <div class="category">
        <h2>{{ $category->name }}</h2>
        
        @foreach ($category->posts as $post)
            {{-- $loop->parent = loop ของ category --}}
            <span>หมวด {{ $loop->parent->iteration }} บทความ {{ $loop->iteration }}</span>
            <p>{{ $post->title }}</p>
        @endforeach
    </div>
@endforeach
```

### @for, @while

```html
<!-- @for -->
@for ($i = 1; $i <= 5; $i++)
    <div class="star {{ $i <= $rating ? 'filled' : '' }}">★</div>
@endfor

<!-- @while -->
@php $i = 0; @endphp
@while ($i < 3)
    <p>รายการที่ {{ $i + 1 }}</p>
    @php $i++; @endphp
@endwhile

<!-- @continue และ @break -->
@foreach ($users as $user)
    @continue($user->banned) {{-- ข้ามถ้า banned --}}
    
    <p>{{ $user->name }}</p>
    
    @break($loop->iteration >= 10) {{-- หยุดเมื่อถึง 10 --}}
@endforeach
```

---

## 4. Template Inheritance

### Layout File

```html
<!-- resources/views/layouts/app.blade.php -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="csrf-token" content="{{ csrf_token() }}">
    
    <title>@yield('title', config('app.name')) | My Blog</title>
    
    {{-- Meta tags --}}
    @yield('meta')
    
    {{-- Favicon --}}
    <link rel="icon" href="{{ asset('favicon.ico') }}">
    
    {{-- Styles --}}
    <link href="https://fonts.googleapis.com/css2?family=Sarabun:wght@400;600;700&display=swap" rel="stylesheet">
    @vite(['resources/css/app.css'])
    
    {{-- Page-specific styles --}}
    @stack('styles')
</head>
<body class="bg-gray-50 font-sarabun">

    {{-- Navigation --}}
    @include('partials.navbar')
    
    {{-- Flash Messages --}}
    @include('partials.flash-messages')
    
    {{-- Main Content --}}
    <main class="container mx-auto px-4 py-8">
        @yield('content')
    </main>
    
    {{-- Footer --}}
    @include('partials.footer')
    
    {{-- Scripts --}}
    @vite(['resources/js/app.js'])
    @stack('scripts')
    
</body>
</html>
```

### Child View

```html
<!-- resources/views/posts/index.blade.php -->
@extends('layouts.app')

@section('title', 'บทความทั้งหมด')

@section('meta')
    <meta name="description" content="รายการบทความทั้งหมดในบล็อกของเรา">
    <meta property="og:title" content="บทความทั้งหมด">
@endsection

@section('content')
    <div class="max-w-4xl mx-auto">
        <h1 class="text-3xl font-bold mb-8">บทความทั้งหมด</h1>
        
        {{-- Search form --}}
        <form method="GET" class="mb-6">
            <input type="search" name="search" 
                   value="{{ request('search') }}"
                   placeholder="ค้นหาบทความ..."
                   class="w-full px-4 py-2 border rounded-lg">
        </form>
        
        {{-- Posts grid --}}
        <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
            @forelse ($posts as $post)
                <article class="bg-white rounded-xl shadow p-6">
                    @if ($post->image)
                        <img src="{{ asset('storage/' . $post->image) }}"
                             alt="{{ $post->title }}"
                             class="w-full h-48 object-cover rounded-lg mb-4">
                    @endif
                    
                    <h2 class="text-xl font-semibold mb-2">
                        <a href="{{ route('posts.show', $post) }}" 
                           class="hover:text-blue-600">
                            {{ $post->title }}
                        </a>
                    </h2>
                    
                    <p class="text-gray-600 text-sm mb-4">{{ $post->excerpt }}</p>
                    
                    <div class="flex items-center justify-between text-sm text-gray-500">
                        <span>{{ $post->author->name }}</span>
                        <span>{{ $post->published_at?->diffForHumans() }}</span>
                    </div>
                </article>
            @empty
                <div class="col-span-2 text-center py-12 text-gray-500">
                    <p class="text-xl">ไม่พบบทความ</p>
                    @auth
                        <a href="{{ route('posts.create') }}" class="text-blue-600 mt-2 block">
                            + สร้างบทความแรก
                        </a>
                    @endauth
                </div>
            @endforelse
        </div>
        
        {{-- Pagination --}}
        <div class="mt-8">
            {{ $posts->links() }}
        </div>
    </div>
@endsection

@push('scripts')
    <script>
        // JavaScript เฉพาะหน้านี้
        document.querySelector('form').addEventListener('submit', function(e) {
            if (!this.search.value.trim()) {
                e.preventDefault();
                window.location.href = '{{ route("posts.index") }}';
            }
        });
    </script>
@endpush
```

### @parent Directive

```html
<!-- Layout มี default content -->
@section('sidebar')
    <h3>Default Sidebar</h3>
    <ul>
        <li>ลิงก์ 1</li>
        <li>ลิงก์ 2</li>
    </ul>
@endsection

<!-- Child extends และเพิ่ม content -->
@section('sidebar')
    @parent {{-- รักษา content เดิม --}}
    <li>ลิงก์เพิ่มเติม</li>
@endsection
```

---

## 5. Blade Components

Components ช่วยสร้าง reusable UI ที่นำไปใช้ซ้ำได้

### สร้าง Component

```bash
# สร้าง Class Component
php artisan make:component Alert
php artisan make:component Card
php artisan make:component Button
php artisan make:component PostCard

# สร้าง Anonymous Component (ไม่มี PHP class)
php artisan make:component forms/input --view
```

### Class Component

```php
// app/View/Components/Alert.php
<?php

namespace App\View\Components;

use Illuminate\View\Component;
use Illuminate\View\View;

class Alert extends Component
{
    public string $icon;
    
    public function __construct(
        public readonly string $type = 'info',
        public readonly string $message = '',
        public readonly bool $dismissible = false,
    ) {
        $this->icon = match($type) {
            'success' => '✓',
            'error'   => '✕',
            'warning' => '⚠',
            default   => 'ℹ',
        };
    }

    public function colorClass(): string
    {
        return match($this->type) {
            'success' => 'bg-green-100 border-green-500 text-green-700',
            'error'   => 'bg-red-100 border-red-500 text-red-700',
            'warning' => 'bg-yellow-100 border-yellow-500 text-yellow-700',
            default   => 'bg-blue-100 border-blue-500 text-blue-700',
        };
    }

    public function render(): View
    {
        return view('components.alert');
    }
}
```

```html
<!-- resources/views/components/alert.blade.php -->
<div class="flex items-start p-4 border-l-4 rounded-r-lg {{ $colorClass() }}" 
     x-data="{ show: true }" 
     x-show="show"
     role="alert">
    
    <span class="text-xl mr-3">{{ $icon }}</span>
    
    <div class="flex-1">
        {{-- $slot = content ที่อยู่ระหว่าง <x-alert>...</x-alert> --}}
        {{ $slot->isNotEmpty() ? $slot : $message }}
    </div>
    
    @if ($dismissible)
        <button @click="show = false" class="ml-auto text-xl leading-none">&times;</button>
    @endif
</div>
```

```html
<!-- การใช้งาน Alert Component -->

{{-- แบบที่ 1: ใช้ attribute --}}
<x-alert type="success" message="บันทึกข้อมูลสำเร็จ!" :dismissible="true" />

{{-- แบบที่ 2: ใช้ slot --}}
<x-alert type="error" :dismissible="true">
    เกิดข้อผิดพลาด: <strong>{{ $error }}</strong>
    <a href="#">ดูรายละเอียด</a>
</x-alert>

{{-- แบบที่ 3: ไม่ระบุ type (ใช้ default) --}}
<x-alert>ข้อมูลทั่วไป</x-alert>
```

### Named Slots

```html
<!-- resources/views/components/card.blade.php -->
<div class="bg-white rounded-xl shadow-md overflow-hidden">
    
    @if ($header->isNotEmpty())
        <div class="px-6 py-4 border-b bg-gray-50">
            {{ $header }}
        </div>
    @endif
    
    <div class="p-6">
        {{ $slot }}
    </div>
    
    @if ($footer->isNotEmpty())
        <div class="px-6 py-4 border-t bg-gray-50 flex justify-end gap-2">
            {{ $footer }}
        </div>
    @endif
</div>
```

```html
<!-- การใช้งาน Card Component กับ Named Slots -->
<x-card>
    <x-slot:header>
        <h3 class="font-semibold text-lg">ชื่อ Card</h3>
    </x-slot:header>
    
    {{-- นี่คือ $slot หลัก --}}
    <p>เนื้อหาของ card</p>
    <p>รายละเอียดเพิ่มเติม...</p>
    
    <x-slot:footer>
        <x-button variant="secondary">ยกเลิก</x-button>
        <x-button variant="primary">บันทึก</x-button>
    </x-slot:footer>
</x-card>
```

### Component ที่ซับซ้อน

```php
// app/View/Components/PostCard.php
<?php

namespace App\View\Components;

use App\Models\Post;
use Illuminate\View\Component;

class PostCard extends Component
{
    public function __construct(
        public readonly Post $post,
        public readonly string $size = 'default', // default, compact, large
        public readonly bool $showAuthor = true,
        public readonly bool $showStats = true,
    ) {}

    public function cardClasses(): string
    {
        return match($this->size) {
            'compact' => 'p-4',
            'large'   => 'p-8',
            default   => 'p-6',
        };
    }

    public function imageClasses(): string
    {
        return match($this->size) {
            'compact' => 'h-32',
            'large'   => 'h-64',
            default   => 'h-48',
        };
    }

    public function render()
    {
        return view('components.post-card');
    }
}
```

```html
<!-- resources/views/components/post-card.blade.php -->
<article class="bg-white rounded-xl shadow hover:shadow-lg transition {{ $cardClasses() }}">
    
    @if ($post->image)
        <img src="{{ asset('storage/' . $post->image) }}"
             alt="{{ $post->title }}"
             class="w-full {{ $imageClasses() }} object-cover rounded-lg mb-4">
    @endif
    
    {{-- Category badge --}}
    @if ($post->category)
        <span class="inline-block px-3 py-1 text-xs bg-blue-100 text-blue-700 rounded-full mb-2">
            {{ $post->category->name }}
        </span>
    @endif
    
    <h2 class="font-semibold {{ $size === 'large' ? 'text-2xl' : 'text-lg' }} mb-2">
        <a href="{{ route('posts.show', $post) }}" class="hover:text-blue-600">
            {{ $post->title }}
        </a>
    </h2>
    
    <p class="text-gray-600 text-sm mb-4">{{ $post->excerpt }}</p>
    
    @if ($showAuthor || $showStats)
        <div class="flex items-center justify-between text-sm text-gray-500">
            @if ($showAuthor)
                <div class="flex items-center gap-2">
                    <img src="{{ $post->author->avatar_url }}" 
                         class="w-6 h-6 rounded-full"
                         alt="{{ $post->author->name }}">
                    {{ $post->author->name }}
                </div>
            @endif
            
            @if ($showStats)
                <div class="flex items-center gap-3">
                    <span>👁 {{ number_format($post->views) }}</span>
                    <span>💬 {{ $post->comments_count ?? 0 }}</span>
                    <span>{{ $post->published_at?->diffForHumans() }}</span>
                </div>
            @endif
        </div>
    @endif
    
    {{-- Extra content จาก slot --}}
    {{ $slot }}
</article>
```

```html
<!-- การใช้งาน PostCard Component -->
@foreach ($posts as $post)
    <x-post-card :post="$post" size="default" :show-author="true" />
@endforeach

{{-- Featured post ขนาดใหญ่ --}}
<x-post-card :post="$featuredPost" size="large">
    <x-slot:default>
        <a href="{{ route('posts.show', $featuredPost) }}" 
           class="mt-4 inline-block btn btn-primary">
            อ่านต่อ →
        </a>
    </x-slot:default>
</x-post-card>
```

### Anonymous Components

```html
<!-- resources/views/components/button.blade.php -->
@props([
    'variant' => 'primary',
    'size' => 'md',
    'type' => 'button',
    'disabled' => false,
])

@php
    $classes = [
        'base' => 'inline-flex items-center justify-center font-medium rounded-lg transition focus:outline-none focus:ring-2',
        'variant' => [
            'primary'   => 'bg-blue-600 text-white hover:bg-blue-700 focus:ring-blue-500',
            'secondary' => 'bg-gray-200 text-gray-700 hover:bg-gray-300 focus:ring-gray-500',
            'danger'    => 'bg-red-600 text-white hover:bg-red-700 focus:ring-red-500',
            'ghost'     => 'bg-transparent text-gray-600 hover:bg-gray-100',
        ][$variant],
        'size' => [
            'sm' => 'px-3 py-1.5 text-sm',
            'md' => 'px-4 py-2',
            'lg' => 'px-6 py-3 text-lg',
        ][$size],
        'disabled' => $disabled ? 'opacity-50 cursor-not-allowed' : '',
    ];
@endphp

<button 
    type="{{ $type }}"
    {{ $disabled ? 'disabled' : '' }}
    {{ $attributes->merge(['class' => implode(' ', $classes)]) }}
>
    {{ $slot }}
</button>
```

```html
<!-- การใช้งาน Button Component -->
<x-button>บันทึก</x-button>
<x-button variant="danger" size="sm">ลบ</x-button>
<x-button variant="secondary" :disabled="$form->isEmpty()">ยกเลิก</x-button>
<x-button type="submit" size="lg" class="w-full">ส่งข้อมูล</x-button>
```

---

## 6. Blade Directives

### @csrf และ @method

```html
<!-- Form สร้างข้อมูล (POST) -->
<form method="POST" action="{{ route('posts.store') }}">
    @csrf {{-- สร้าง hidden input: _token --}}
    <!-- form fields -->
    <button type="submit">บันทึก</button>
</form>

<!-- Form แก้ไขข้อมูล (PUT) -->
<form method="POST" action="{{ route('posts.update', $post) }}">
    @csrf
    @method('PUT') {{-- สร้าง hidden input: _method = PUT --}}
    <!-- form fields -->
</form>

<!-- Form ลบข้อมูล (DELETE) -->
<form method="POST" action="{{ route('posts.destroy', $post) }}">
    @csrf
    @method('DELETE')
    <button type="submit">ลบ</button>
</form>
```

### @include

```html
<!-- Include view อื่น -->
@include('partials.navbar')

{{-- Include พร้อมส่ง data --}}
@include('partials.post-card', ['post' => $post, 'showImage' => true])

{{-- Include ถ้า file มีอยู่ --}}
@includeIf('partials.debug-bar')

{{-- Include ตามเงื่อนไข --}}
@includeWhen(auth()->check(), 'partials.user-menu')
@includeUnless(auth()->check(), 'partials.guest-menu')

{{-- Include หนึ่งในหลาย views (ใช้อันแรกที่เจอ) --}}
@includeFirst(['custom.navbar', 'partials.navbar'])
```

### @each

```html
{{-- วน Loop พร้อม include view --}}
{{-- @each('view', $collection, 'variable', 'empty-view') --}}
@each('partials.post-item', $posts, 'post', 'partials.no-posts')

{{-- เทียบเท่ากับ --}}
@forelse ($posts as $post)
    @include('partials.post-item', compact('post'))
@empty
    @include('partials.no-posts')
@endforelse
```

### @error

```html
<!-- แสดง validation error -->
<input type="text" name="title" 
       value="{{ old('title') }}"
       class="border rounded px-3 py-2 @error('title') border-red-500 @enderror">

@error('title')
    <p class="text-red-500 text-sm mt-1">{{ $message }}</p>
@enderror

<!-- หรือสั้นกว่า -->
<p class="text-red-500">{{ $errors->first('title') }}</p>

{{-- แสดง errors ทั้งหมด --}}
@if ($errors->any())
    <div class="bg-red-100 p-4 rounded mb-4">
        <ul class="list-disc pl-5">
            @foreach ($errors->all() as $error)
                <li class="text-red-700">{{ $error }}</li>
            @endforeach
        </ul>
    </div>
@endif
```

### @stack และ @push

```html
<!-- Layout -->
<head>
    @stack('styles')  {{-- Placeholder --}}
</head>
<body>
    @yield('content')
    @stack('scripts') {{-- Placeholder --}}
</body>

<!-- Child View หรือ Component -->
@push('styles')
    <link rel="stylesheet" href="{{ asset('css/chart.css') }}">
    <style>
        .custom-class { color: red; }
    </style>
@endpush

@push('scripts')
    <script src="{{ asset('js/chart.js') }}"></script>
    <script>
        new Chart(document.getElementById('myChart'), {...});
    </script>
@endpush

{{-- @prepend - เพิ่มก่อน (ตรงข้ามกับ @push) --}}
@prepend('scripts')
    <script>console.log('This runs first');</script>
@endprepend
```

---

## 7. Custom Directives

### สร้าง Custom Directive

```php
// app/Providers/AppServiceProvider.php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\Blade;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // @money directive
        Blade::directive('money', function ($amount) {
            return "<?php echo '฿' . number_format({$amount}, 2); ?>";
        });

        // @datetime directive
        Blade::directive('datetime', function ($expression) {
            return "<?php echo \Carbon\Carbon::parse({$expression})->format('d เดือน m ปี Y เวลา H:i น.'); ?>";
        });

        // @role directive สำหรับ authorization
        Blade::directive('role', function ($role) {
            return "<?php if (auth()->check() && auth()->user()->hasRole({$role})): ?>";
        });

        Blade::directive('endrole', function () {
            return "<?php endif; ?>";
        });

        // @admin shorthand
        Blade::directive('admin', function () {
            return "<?php if (auth()->check() && auth()->user()->isAdmin()): ?>";
        });

        Blade::directive('endadmin', function () {
            return "<?php endif; ?>";
        });

        // @truncate directive
        Blade::directive('truncate', function ($expression) {
            [$text, $length, $suffix] = array_pad(
                array_map('trim', explode(',', $expression, 3)),
                3,
                null
            );
            $length = $length ?? 100;
            $suffix = $suffix ?? "'...'";
            return "<?php echo \Illuminate\Support\Str::limit({$text}, {$length}, {$suffix}); ?>";
        });
    }
}
```

### การใช้งาน Custom Directives

```html
<!-- @money -->
<p>ราคา: @money($product->price)</p>
{{-- Output: ราคา: ฿1,299.00 --}}

<!-- @datetime -->
<p>เผยแพร่: @datetime($post->published_at)</p>
{{-- Output: เผยแพร่: 15 เดือน 10 ปี 2567 เวลา 09:30 น. --}}

<!-- @role -->
@role('editor')
    <a href="{{ route('posts.create') }}">สร้างบทความ</a>
@endrole

<!-- @admin -->
@admin
    <a href="{{ route('admin.dashboard') }}">Admin Panel</a>
@endadmin

<!-- @truncate -->
<p>@truncate($post->content, 200)</p>
<p>@truncate($post->content, 100, '... อ่านต่อ')</p>
```

### Custom If Directive

```php
// app/Providers/AppServiceProvider.php
public function boot(): void
{
    // @subscribed
    Blade::if('subscribed', function ($plan = null) {
        if (!auth()->check()) return false;
        
        $user = auth()->user();
        
        if ($plan) {
            return $user->subscribed($plan);
        }
        
        return $user->hasActiveSubscription();
    });

    // @premium
    Blade::if('premium', function () {
        return auth()->check() && auth()->user()->isPremium();
    });

    // @mobile (ตรวจสอบ user agent)
    Blade::if('mobile', function () {
        return preg_match('/Mobile|Android|iPhone/i', request()->userAgent());
    });
}
```

```html
<!-- @subscribed -->
@subscribed
    <p>ขอบคุณที่เป็นสมาชิก Premium!</p>
@elsesubscribed
    <a href="/upgrade">อัปเกรดเพื่อใช้ฟีเจอร์เพิ่มเติม</a>
@endsubscribed

@subscribed('pro')
    <p>สมาชิก Pro Plan</p>
@endsubscribed

<!-- @premium -->
@premium
    <div class="premium-content">เนื้อหา exclusive สำหรับสมาชิก Premium</div>
@elsepremium
    <div class="upgrade-cta">
        <p>อัปเกรดเพื่อดูเนื้อหานี้</p>
        <x-button variant="primary">อัปเกรดตอนนี้</x-button>
    </div>
@endpremium

<!-- @mobile -->
@mobile
    <div class="mobile-banner">กำลังใช้งานบน Mobile</div>
@endmobile
```

---

## Workshop: สร้าง Layout สำหรับ Blog

### โจทย์

สร้าง Layout system ที่มี:
1. Main layout ที่ responsive
2. Admin layout แยกต่างหาก
3. Components: Navbar, Footer, Sidebar, PostCard, Pagination

### Step 1: Main Layout

```html
<!-- resources/views/layouts/app.blade.php -->
<!DOCTYPE html>
<html lang="th" class="scroll-smooth">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="csrf-token" content="{{ csrf_token() }}">
    
    <title>@yield('title', 'หน้าแรก') | {{ config('app.name') }}</title>
    
    @yield('meta')
    
    <!-- Open Graph -->
    <meta property="og:site_name" content="{{ config('app.name') }}">
    <meta property="og:title" content="@yield('og:title', config('app.name'))">
    <meta property="og:description" content="@yield('og:description', '')">
    @hasSection('og:image')
        <meta property="og:image" content="@yield('og:image')">
    @endif
    
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link href="https://fonts.googleapis.com/css2?family=Sarabun:wght@300;400;500;600;700&display=swap" rel="stylesheet">
    
    @vite(['resources/css/app.css'])
    @stack('styles')
</head>
<body class="min-h-screen flex flex-col bg-gray-50 font-sarabun text-gray-900">
    
    <!-- Navbar -->
    <x-navbar />
    
    <!-- Flash Messages -->
    <div class="container mx-auto px-4 pt-4">
        @foreach (['success', 'error', 'warning', 'info'] as $type)
            @if (session($type))
                <x-alert :type="$type" :message="session($type)" :dismissible="true" class="mb-2" />
            @endif
        @endforeach
    </div>
    
    <!-- Content -->
    <main class="flex-1 container mx-auto px-4 py-8">
        @hasSection('sidebar')
            <div class="grid grid-cols-1 lg:grid-cols-4 gap-8">
                <div class="lg:col-span-3">
                    @yield('content')
                </div>
                <aside class="lg:col-span-1">
                    @yield('sidebar')
                </aside>
            </div>
        @else
            @yield('content')
        @endif
    </main>
    
    <!-- Footer -->
    <x-footer />
    
    @vite(['resources/js/app.js'])
    @stack('scripts')
</body>
</html>
```

### Step 2: Navbar Component

```php
// app/View/Components/Navbar.php
<?php

namespace App\View\Components;

use Illuminate\View\Component;

class Navbar extends Component
{
    public function render()
    {
        return view('components.navbar');
    }
}
```

```html
<!-- resources/views/components/navbar.blade.php -->
<nav class="bg-white shadow-sm sticky top-0 z-50">
    <div class="container mx-auto px-4">
        <div class="flex items-center justify-between h-16">
            
            <!-- Logo -->
            <a href="{{ route('home') }}" class="flex items-center gap-2 font-bold text-xl text-blue-600">
                <svg class="w-8 h-8" fill="currentColor" viewBox="0 0 20 20">
                    <path d="M2 5a2 2 0 012-2h7a2 2 0 012 2v4a2 2 0 01-2 2H9l-3 3v-3H4a2 2 0 01-2-2V5z"/>
                </svg>
                {{ config('app.name') }}
            </a>
            
            <!-- Desktop Navigation -->
            <div class="hidden md:flex items-center gap-6">
                <a href="{{ route('posts.index') }}" 
                   class="text-gray-600 hover:text-blue-600 transition {{ request()->routeIs('posts.*') ? 'text-blue-600 font-semibold' : '' }}">
                    บทความ
                </a>
                <a href="{{ route('categories.index') }}"
                   class="text-gray-600 hover:text-blue-600 transition {{ request()->routeIs('categories.*') ? 'text-blue-600 font-semibold' : '' }}">
                    หมวดหมู่
                </a>
                <a href="{{ route('about') }}"
                   class="text-gray-600 hover:text-blue-600 transition">
                    เกี่ยวกับ
                </a>
            </div>
            
            <!-- Auth Links -->
            <div class="flex items-center gap-4">
                @auth
                    <div class="relative" x-data="{ open: false }">
                        <button @click="open = !open" class="flex items-center gap-2 focus:outline-none">
                            <img src="{{ auth()->user()->avatar_url }}" 
                                 class="w-8 h-8 rounded-full border"
                                 alt="{{ auth()->user()->name }}">
                            <span class="hidden md:block text-sm">{{ auth()->user()->name }}</span>
                        </button>
                        
                        <div x-show="open" @click.away="open = false"
                             class="absolute right-0 mt-2 w-48 bg-white rounded-xl shadow-lg py-1 border">
                            <a href="{{ route('profile') }}" class="block px-4 py-2 text-sm hover:bg-gray-50">โปรไฟล์</a>
                            <a href="{{ route('my.posts.index') }}" class="block px-4 py-2 text-sm hover:bg-gray-50">บทความของฉัน</a>
                            <a href="{{ route('posts.create') }}" class="block px-4 py-2 text-sm hover:bg-gray-50">+ สร้างบทความ</a>
                            @admin
                                <hr class="my-1">
                                <a href="{{ route('admin.dashboard') }}" class="block px-4 py-2 text-sm text-blue-600 hover:bg-gray-50">Admin Panel</a>
                            @endadmin
                            <hr class="my-1">
                            <form method="POST" action="{{ route('logout') }}">
                                @csrf
                                <button type="submit" class="w-full text-left px-4 py-2 text-sm text-red-600 hover:bg-gray-50">
                                    ออกจากระบบ
                                </button>
                            </form>
                        </div>
                    </div>
                @else
                    <a href="{{ route('login') }}" class="text-gray-600 hover:text-blue-600">เข้าสู่ระบบ</a>
                    <x-button variant="primary" size="sm">
                        <a href="{{ route('register') }}" class="text-white">สมัครสมาชิก</a>
                    </x-button>
                @endauth
            </div>
        </div>
    </div>
</nav>
```

### Step 3: Blog Post View

```html
<!-- resources/views/posts/show.blade.php -->
@extends('layouts.app')

@section('title', $post->title)

@section('meta')
    <meta name="description" content="{{ $post->excerpt }}">
@endsection

@section('og:title', $post->title)
@section('og:description', $post->excerpt)
@if ($post->image)
    @section('og:image', asset('storage/' . $post->image))
@endif

@section('content')
    <article class="max-w-3xl mx-auto">
        
        {{-- Header --}}
        <header class="mb-8">
            @if ($post->category)
                <a href="{{ route('categories.show', $post->category) }}"
                   class="inline-block px-3 py-1 bg-blue-100 text-blue-700 rounded-full text-sm mb-4">
                    {{ $post->category->name }}
                </a>
            @endif
            
            <h1 class="text-4xl font-bold mb-4 leading-tight">{{ $post->title }}</h1>
            
            <div class="flex items-center gap-4 text-gray-500 text-sm">
                <div class="flex items-center gap-2">
                    <img src="{{ $post->author->avatar_url }}" class="w-8 h-8 rounded-full" alt="">
                    <a href="{{ route('users.show', $post->author) }}" class="hover:text-blue-600">
                        {{ $post->author->name }}
                    </a>
                </div>
                <span>•</span>
                <time datetime="{{ $post->published_at->toISOString() }}">
                    {{ $post->published_at->format('d M Y') }}
                </time>
                <span>•</span>
                <span>{{ $post->read_time }} นาทีในการอ่าน</span>
                <span>•</span>
                <span>👁 {{ number_format($post->views) }} ครั้ง</span>
            </div>
        </header>
        
        {{-- Featured Image --}}
        @isset($post->image)
            <img src="{{ asset('storage/' . $post->image) }}"
                 alt="{{ $post->title }}"
                 class="w-full rounded-xl mb-8 shadow-md">
        @endisset
        
        {{-- Content --}}
        <div class="prose prose-lg max-w-none mb-8">
            {!! $post->content !!}
        </div>
        
        {{-- Tags --}}
        @if ($post->tags->isNotEmpty())
            <div class="flex flex-wrap gap-2 mb-8">
                @foreach ($post->tags as $tag)
                    <a href="{{ route('tags.show', $tag) }}"
                       class="px-3 py-1 bg-gray-100 hover:bg-gray-200 rounded-full text-sm">
                        #{{ $tag->name }}
                    </a>
                @endforeach
            </div>
        @endif
        
        {{-- Author Actions --}}
        @can('update', $post)
            <div class="flex gap-3 mb-8 p-4 bg-blue-50 rounded-xl">
                <a href="{{ route('posts.edit', $post) }}" class="btn btn-secondary">แก้ไข</a>
                <form method="POST" action="{{ route('posts.publish', $post) }}">
                    @csrf
                    @method('PATCH')
                    <button type="submit" class="btn {{ $post->status === 'published' ? 'btn-warning' : 'btn-success' }}">
                        {{ $post->status === 'published' ? 'ซ่อนบทความ' : 'เผยแพร่' }}
                    </button>
                </form>
                <form method="POST" action="{{ route('posts.destroy', $post) }}">
                    @csrf
                    @method('DELETE')
                    <button type="submit" class="btn btn-danger"
                            onclick="return confirm('ลบบทความนี้?')">ลบ</button>
                </form>
            </div>
        @endcan
        
        {{-- Comments --}}
        <section id="comments">
            <h3 class="text-2xl font-semibold mb-6">
                ความคิดเห็น ({{ $post->comments->count() }})
            </h3>
            
            @auth
                <form method="POST" action="{{ route('comments.store', $post) }}" class="mb-8">
                    @csrf
                    <textarea name="content" rows="4"
                              class="w-full border rounded-xl p-4 focus:ring-2 focus:ring-blue-500"
                              placeholder="แสดงความคิดเห็น...">{{ old('content') }}</textarea>
                    @error('content')
                        <p class="text-red-500 text-sm mt-1">{{ $message }}</p>
                    @enderror
                    <x-button type="submit" class="mt-2">ส่งความคิดเห็น</x-button>
                </form>
            @else
                <p class="mb-8 text-gray-500">
                    <a href="{{ route('login') }}" class="text-blue-600">เข้าสู่ระบบ</a> เพื่อแสดงความคิดเห็น
                </p>
            @endauth
            
            <div class="space-y-4">
                @forelse ($post->comments as $comment)
                    <div class="bg-white rounded-xl p-4 shadow-sm">
                        <div class="flex items-center justify-between mb-2">
                            <div class="flex items-center gap-2">
                                <img src="{{ $comment->author->avatar_url }}" class="w-8 h-8 rounded-full">
                                <strong>{{ $comment->author->name }}</strong>
                            </div>
                            <div class="flex items-center gap-2">
                                <span class="text-gray-400 text-sm">{{ $comment->created_at->diffForHumans() }}</span>
                                @can('delete', $comment)
                                    <form method="POST" action="{{ route('comments.destroy', $comment) }}">
                                        @csrf @method('DELETE')
                                        <button type="submit" class="text-red-500 text-sm hover:underline">ลบ</button>
                                    </form>
                                @endcan
                            </div>
                        </div>
                        <p class="text-gray-700">{{ $comment->content }}</p>
                    </div>
                @empty
                    <p class="text-gray-500 text-center py-8">ยังไม่มีความคิดเห็น เป็นคนแรกที่แสดงความคิดเห็น!</p>
                @endforelse
            </div>
        </section>
    </article>
    
    {{-- Related Posts --}}
    @if ($relatedPosts->isNotEmpty())
        <div class="max-w-3xl mx-auto mt-12">
            <h3 class="text-2xl font-semibold mb-6">บทความที่เกี่ยวข้อง</h3>
            <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                @foreach ($relatedPosts as $related)
                    <x-post-card :post="$related" size="compact" />
                @endforeach
            </div>
        </div>
    @endif
@endsection

@section('sidebar')
    <div class="space-y-6">
        <!-- About Author -->
        <x-card>
            <x-slot:header>เกี่ยวกับผู้เขียน</x-slot:header>
            <div class="text-center">
                <img src="{{ $post->author->avatar_url }}" class="w-20 h-20 rounded-full mx-auto mb-3">
                <h4 class="font-semibold">{{ $post->author->name }}</h4>
                <p class="text-gray-500 text-sm mt-1">{{ $post->author->bio }}</p>
                <a href="{{ route('users.show', $post->author) }}" class="text-blue-600 text-sm mt-2 inline-block">
                    ดูบทความทั้งหมด →
                </a>
            </div>
        </x-card>
        
        <!-- Popular Posts -->
        <x-card>
            <x-slot:header>บทความยอดนิยม</x-slot:header>
            @foreach (\App\Models\Post::published()->orderByDesc('views')->limit(5)->get() as $popular)
                <div class="mb-3 pb-3 border-b last:border-0">
                    <a href="{{ route('posts.show', $popular) }}" 
                       class="text-sm hover:text-blue-600 line-clamp-2">
                        {{ $popular->title }}
                    </a>
                    <p class="text-xs text-gray-400 mt-1">{{ number_format($popular->views) }} ครั้ง</p>
                </div>
            @endforeach
        </x-card>
    </div>
@endsection
```

---

## Quiz

### คำถาม

**1.** `{!! $content !!}` ต่างจาก `{{ $content }}` อย่างไร?

a) `{!! !!}` escape HTML, `{{ }}` ไม่ escape
b) `{{ }}` escape HTML (XSS-safe), `{!! !!}` ไม่ escape
c) ทั้งคู่เหมือนกัน
d) `{!! !!}` ใช้กับ arrays, `{{ }}` ใช้กับ strings

**2.** `@forelse` ต่างจาก `@foreach` อย่างไร?

a) `@forelse` เร็วกว่า
b) `@forelse` มี `@empty` block สำหรับเมื่อ collection ว่าง
c) `@forelse` ใช้กับ arrays เท่านั้น
d) `@forelse` ไม่มี `$loop` variable

**3.** ใน Blade component, `{{ $slot }}` คืออะไร?

a) ตัวแปรที่ส่งมาใน attribute
b) content ที่อยู่ระหว่าง opening และ closing component tag
c) default content ของ component
d) method ของ Component class

**4.** `@stack('scripts')` และ `@push('scripts')` ทำงานร่วมกันอย่างไร?

a) `@stack` สร้าง variable, `@push` append string
b) `@stack` เป็น placeholder ใน layout, `@push` ใน child views เพิ่ม content เข้าไป
c) ทั้งคู่ต้องอยู่ใน layout เท่านั้น
d) `@push` ต้องอยู่ก่อน `@stack`

**5.** `$loop->iteration` กับ `$loop->index` ต่างกันอย่างไร?

a) เหมือนกัน
b) `iteration` เริ่มจาก 1, `index` เริ่มจาก 0
c) `iteration` เริ่มจาก 0, `index` เริ่มจาก 1
d) `index` นับถอยหลัง

### เฉลย

| ข้อ | คำตอบ | อธิบาย |
|-----|-------|--------|
| 1 | **b** | `{{ }}` escape HTML ป้องกัน XSS, `{!! !!}` แสดง raw HTML |
| 2 | **b** | `@forelse` มี `@empty` รองรับกรณี collection ว่าง |
| 3 | **b** | `$slot` คือ content ระหว่าง `<x-component>...</x-component>` |
| 4 | **b** | `@stack` เป็น placeholder, `@push` inject content จากที่ต่างๆ |
| 5 | **b** | `$loop->iteration` นับจาก 1, `$loop->index` นับจาก 0 |

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- Blade syntax: echoing, conditionals, loops ทุกรูปแบบ
- Template Inheritance ด้วย `@extends`, `@section`, `@yield`
- Blade Components ทั้ง Class-based และ Anonymous
- Named Slots สำหรับ flexible components
- Built-in directives: @csrf, @method, @include, @stack, @push, @error
- Custom Directives สร้าง shorthand syntax เอง
- Workshop: Layout สมบูรณ์สำหรับ Blog

---

## ลิงก์ไป Part ถัดไป

➡️ **[Part 030: Laravel Migrations](./part-030-laravel-migrations.md)**

ใน Part ถัดไปเราจะเรียนรู้ Database Migrations, Column types ทั้งหมด, Foreign Keys, Seeders & Factories และสร้าง Database schema สำหรับ E-commerce
