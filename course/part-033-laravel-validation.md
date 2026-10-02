# Part 033: Laravel Validation

## ระดับ: Intermediate
## เวลาที่ใช้: 3-4 ชั่วโมง

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้างและใช้ Form Request Validation
- รู้จัก Validation Rules ที่สำคัญทั้งหมด
- สร้าง Custom Validation Rules
- กำหนด Error Messages แบบ custom
- ทำ Conditional Validation
- สร้างระบบ User Registration Form Validation ที่สมบูรณ์

---

## 1. พื้นฐาน Validation

### 1.1 Inline Validation ใน Controller

```php
<?php
// app/Http/Controllers/UserController.php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Validation\ValidationException;

class UserController extends Controller
{
    public function store(Request $request)
    {
        // วิธีง่ายที่สุด: validate() ใน Request
        $validated = $request->validate([
            'name'  => 'required|string|max:255',
            'email' => 'required|email|unique:users',
            'age'   => 'required|integer|min:18',
        ]);

        // ถ้า validation ผ่าน จะได้ $validated array
        // ถ้าไม่ผ่าน จะ redirect กลับพร้อม errors อัตโนมัติ
        
        User::create($validated);
        
        return redirect()->route('users.index')
            ->with('success', 'สร้างผู้ใช้สำเร็จ');
    }
}
```

### 1.2 Validator Facade

```php
<?php
use Illuminate\Support\Facades\Validator;

// สร้าง Validator แบบ manual
$validator = Validator::make($request->all(), [
    'name'  => 'required|string|max:255',
    'email' => 'required|email|unique:users',
]);

// เช็คว่า fail
if ($validator->fails()) {
    return redirect()->back()
        ->withErrors($validator)
        ->withInput();
}

// ดึง validated data
$validated = $validator->validated();

// ดึง errors
$errors = $validator->errors();
$nameErrors = $validator->errors()->get('name'); // Array ของ errors สำหรับ field 'name'
$firstNameError = $validator->errors()->first('name'); // Error แรกของ field 'name'
$allErrors = $validator->errors()->all(); // ทุก errors เป็น flat array
```

---

## 2. Form Request Validation

Form Request คือวิธีที่ดีที่สุดสำหรับ validation ที่ซับซ้อน

### 2.1 สร้าง Form Request

```bash
# สร้าง Form Request
php artisan make:request StoreUserRequest
php artisan make:request UpdateUserRequest
```

### 2.2 โครงสร้าง Form Request

```php
<?php
// app/Http/Requests/StoreUserRequest.php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Contracts\Validation\Validator;
use Illuminate\Http\Exceptions\HttpResponseException;

class StoreUserRequest extends FormRequest
{
    /**
     * กำหนดว่าใครมีสิทธิ์ส่ง request นี้
     * return true = ทุกคนมีสิทธิ์
     */
    public function authorize(): bool
    {
        return true;
        
        // ตัวอย่างการ authorize ตาม role
        // return $this->user()->can('create', User::class);
    }

    /**
     * Validation rules
     */
    public function rules(): array
    {
        return [
            'name'                  => 'required|string|min:2|max:255',
            'email'                 => 'required|email:rfc,dns|unique:users,email',
            'password'              => 'required|string|min:8|confirmed',
            'password_confirmation' => 'required',
            'phone'                 => 'nullable|string|regex:/^[0-9]{10}$/',
            'birthday'              => 'nullable|date|before:today|after:1900-01-01',
            'role'                  => 'required|in:admin,editor,user',
            'avatar'                => 'nullable|image|mimes:jpeg,png,webp|max:2048',
            'terms'                 => 'required|accepted',
        ];
    }

    /**
     * Custom error messages
     */
    public function messages(): array
    {
        return [
            'name.required'         => 'กรุณากรอกชื่อ',
            'name.min'              => 'ชื่อต้องมีอย่างน้อย :min ตัวอักษร',
            'name.max'              => 'ชื่อต้องไม่เกิน :max ตัวอักษร',
            'email.required'        => 'กรุณากรอกอีเมล',
            'email.email'           => 'รูปแบบอีเมลไม่ถูกต้อง',
            'email.unique'          => 'อีเมลนี้ถูกใช้งานแล้ว',
            'password.required'     => 'กรุณากรอกรหัสผ่าน',
            'password.min'          => 'รหัสผ่านต้องมีอย่างน้อย :min ตัวอักษร',
            'password.confirmed'    => 'รหัสผ่านไม่ตรงกัน',
            'phone.regex'           => 'เบอร์โทรต้องเป็นตัวเลข 10 หลัก',
            'birthday.before'       => 'วันเกิดต้องน้อยกว่าวันนี้',
            'role.in'               => 'Role ไม่ถูกต้อง',
            'avatar.image'          => 'ไฟล์ต้องเป็นรูปภาพ',
            'avatar.max'            => 'รูปภาพต้องมีขนาดไม่เกิน 2MB',
            'terms.accepted'        => 'กรุณายอมรับเงื่อนไขการใช้งาน',
        ];
    }

    /**
     * Custom attribute names (ใน error messages)
     */
    public function attributes(): array
    {
        return [
            'name'     => 'ชื่อ',
            'email'    => 'อีเมล',
            'password' => 'รหัสผ่าน',
            'phone'    => 'เบอร์โทร',
            'birthday' => 'วันเกิด',
            'role'     => 'บทบาท',
            'avatar'   => 'รูปโปรไฟล์',
            'terms'    => 'เงื่อนไขการใช้งาน',
        ];
    }

    /**
     * Prepare data ก่อน validate
     */
    protected function prepareForValidation(): void
    {
        $this->merge([
            'email' => strtolower($this->email),
            'name'  => trim($this->name),
        ]);
    }

    /**
     * Custom response เมื่อ validation ล้มเหลว
     * (ปกติไม่ต้องกำหนด Laravel จัดการให้)
     */
    // protected function failedValidation(Validator $validator): void
    // {
    //     throw new HttpResponseException(
    //         response()->json([
    //             'message' => 'Validation failed',
    //             'errors'  => $validator->errors(),
    //         ], 422)
    //     );
    // }
}
```

### 2.3 ใช้ Form Request ใน Controller

```php
<?php
// app/Http/Controllers/UserController.php

namespace App\Http\Controllers;

use App\Http\Requests\StoreUserRequest;
use App\Http\Requests\UpdateUserRequest;
use App\Models\User;

class UserController extends Controller
{
    public function store(StoreUserRequest $request)
    {
        // ถ้า method นี้ทำงาน แปลว่า validation ผ่านแล้ว
        // $request->validated() คืน array ของ validated data เท่านั้น
        $validated = $request->validated();

        $user = User::create($validated);

        return redirect()->route('users.show', $user)
            ->with('success', 'สร้างผู้ใช้สำเร็จ');
    }

    public function update(UpdateUserRequest $request, User $user)
    {
        $user->update($request->validated());

        return redirect()->route('users.show', $user)
            ->with('success', 'อัพเดตผู้ใช้สำเร็จ');
    }
}
```

---

## 3. Validation Rules ที่สำคัญ

### 3.1 Basic Rules

```php
<?php
$rules = [
    // Required
    'field'     => 'required',           // ต้องมีค่า (ไม่ใช่ null, empty string, empty array)
    'field'     => 'required_if:other,value', // required ถ้า other field == value
    'field'     => 'required_unless:other,value', // required ถ้า other field != value
    'field'     => 'required_with:a,b',  // required ถ้ามี a หรือ b
    'field'     => 'required_with_all:a,b', // required ถ้ามีทั้ง a และ b
    'field'     => 'required_without:a', // required ถ้าไม่มี a
    'field'     => 'nullable',           // อนุญาตให้เป็น null

    // String
    'field'     => 'string',
    'field'     => 'min:5',              // ความยาวขั้นต่ำ
    'field'     => 'max:255',            // ความยาวสูงสุด
    'field'     => 'size:10',            // ความยาวต้องเท่ากับ 10
    'field'     => 'between:5,20',       // ความยาวอยู่ระหว่าง 5-20
    'field'     => 'alpha',              // เฉพาะ a-z, A-Z
    'field'     => 'alpha_num',          // a-z, A-Z, 0-9
    'field'     => 'alpha_dash',         // a-z, A-Z, 0-9, -, _
    'field'     => 'alpha_num_unicode',  // รวม unicode characters
    'field'     => 'lowercase',          // ต้องเป็นตัวพิมพ์เล็กทั้งหมด
    'field'     => 'uppercase',          // ต้องเป็นตัวพิมพ์ใหญ่ทั้งหมด
    'field'     => 'starts_with:foo,bar', // ต้องขึ้นต้นด้วย foo หรือ bar
    'field'     => 'ends_with:foo,bar',  // ต้องลงท้ายด้วย foo หรือ bar
    'field'     => 'doesnt_start_with:foo', // ต้องไม่ขึ้นต้นด้วย foo
    'field'     => 'doesnt_end_with:foo',   // ต้องไม่ลงท้ายด้วย foo
    'field'     => 'not_regex:/pattern/',   // ต้องไม่ match pattern

    // Numeric
    'field'     => 'integer',
    'field'     => 'numeric',
    'field'     => 'decimal:2',          // ทศนิยม 2 ตำแหน่ง
    'field'     => 'min:0',              // ค่าขั้นต่ำ
    'field'     => 'max:100',            // ค่าสูงสุด
    'field'     => 'gt:other_field',     // มากกว่า field อื่น
    'field'     => 'gte:other_field',    // มากกว่าหรือเท่ากับ
    'field'     => 'lt:other_field',     // น้อยกว่า
    'field'     => 'lte:other_field',    // น้อยกว่าหรือเท่ากับ
    'field'     => 'multiple_of:5',      // ต้องหาร 5 ลงตัว

    // Boolean
    'field'     => 'boolean',            // true, false, 1, 0, "1", "0"
    'field'     => 'accepted',           // "yes", "on", 1, "1", true, "true"
    'field'     => 'declined',           // "no", "off", 0, "0", false, "false"
];
```

### 3.2 Date & Time Rules

```php
<?php
$rules = [
    'date'         => 'date',                          // วันที่ถูกต้อง
    'date'         => 'date_format:Y-m-d',             // รูปแบบ 2024-01-15
    'date'         => 'date_format:d/m/Y',             // รูปแบบ 15/01/2024
    'birthday'     => 'before:today',                  // ก่อนวันนี้
    'birthday'     => 'before:2000-01-01',             // ก่อนวันที่กำหนด
    'start_date'   => 'after:yesterday',               // หลังวานนี้
    'end_date'     => 'after:start_date',              // หลัง start_date
    'event_date'   => 'after_or_equal:today',          // วันนี้หรือหลัง
    'expire_date'  => 'before_or_equal:2025-12-31',   // ก่อนหรือเท่ากับ
];
```

### 3.3 Database Rules

```php
<?php
$rules = [
    // Unique: ต้องไม่ซ้ำใน table
    'email'    => 'unique:users',              // unique ใน table users คอลัมน์ email
    'email'    => 'unique:users,email',        // ระบุชื่อคอลัมน์
    'email'    => 'unique:users,email,5',      // ยกเว้น id = 5 (สำหรับ update)
    
    // Unique แบบ object (ยืดหยุ่นกว่า)
    'email'    => Rule::unique('users')->ignore($user->id),
    'email'    => Rule::unique('users')->where(fn ($q) => $q->where('role', 'admin')),

    // Exists: ต้องมีอยู่ใน table
    'user_id'  => 'exists:users,id',
    'country'  => 'exists:countries,code',

    // Exists แบบ object
    'category_id' => Rule::exists('categories', 'id')->where('is_active', true),
];
```

### 3.4 File Rules

```php
<?php
$rules = [
    'file'     => 'file',                      // ต้องเป็นไฟล์
    'image'    => 'image',                     // jpeg, png, gif, bmp, svg, webp
    'avatar'   => 'mimes:jpeg,png,webp',       // เฉพาะ mime types
    'document' => 'mimetypes:application/pdf', // เฉพาะ mime type
    'file'     => 'max:10240',                 // ขนาดสูงสุด (KB) = 10MB
    'file'     => 'min:1',                     // ขนาดขั้นต่ำ (KB)
    'image'    => 'dimensions:min_width=100,min_height=100', // ขนาดรูป
    'image'    => 'dimensions:ratio=16/9',     // อัตราส่วน
    'files'    => 'array|max:5',               // ไม่เกิน 5 files
    'files.*'  => 'file|max:2048',             // แต่ละ file ไม่เกิน 2MB
];
```

### 3.5 Array Rules

```php
<?php
$rules = [
    'items'         => 'required|array|min:1|max:10',  // array ที่มี 1-10 items
    'items.*'       => 'required|integer|exists:products,id', // แต่ละ item
    'tags'          => 'array',
    'tags.*'        => 'string|max:50',
    'settings'      => 'array',
    'settings.theme'=> 'in:light,dark',
    'settings.lang' => 'in:th,en',
    
    // distinct: ไม่มีค่าซ้ำใน array
    'roles'         => 'array|distinct',
    'roles.*'       => 'string|in:admin,editor,user',
];
```

### 3.6 Special Rules

```php
<?php
use Illuminate\Validation\Rule;
use Illuminate\Validation\Rules\Password;

$rules = [
    // In / Not In
    'status'   => 'in:active,inactive,pending',
    'status'   => Rule::in(['active', 'inactive', 'pending']),
    'color'    => 'not_in:red,blue',

    // Regex
    'phone'    => 'regex:/^[0-9]{10}$/',
    'username' => 'regex:/^[a-z0-9_]{3,20}$/',

    // Confirmed: ต้องมี field_confirmation
    'password' => 'confirmed',     // จะเช็คกับ password_confirmation
    'email'    => 'confirmed',     // จะเช็คกับ email_confirmation

    // Same: ต้องเหมือนกับ field อื่น
    'password' => 'same:password_confirmation',

    // Different: ต้องต่างจาก field อื่น  
    'new_password' => 'different:old_password',

    // IP Address
    'ip_address' => 'ip',          // IPv4 หรือ IPv6
    'ip_address' => 'ipv4',        // เฉพาะ IPv4
    'ip_address' => 'ipv6',        // เฉพาะ IPv6

    // URL
    'website'  => 'url',
    'website'  => 'url:http,https',
    'website'  => 'active_url',    // เช็คว่า URL ใช้งานได้จริง

    // UUID
    'uuid'     => 'uuid',

    // JSON
    'metadata' => 'json',

    // Password Rules (Laravel 10+)
    'password' => [
        'required',
        'confirmed',
        Password::min(8)
            ->letters()          // ต้องมีตัวอักษร
            ->mixedCase()        // ตัวพิมพ์ใหญ่และเล็ก
            ->numbers()          // ต้องมีตัวเลข
            ->symbols()          // ต้องมีสัญลักษณ์
            ->uncompromised(),   // เช็คใน haveibeenpwned.com
    ],
];
```

---

## 4. Custom Validation Rules

### 4.1 สร้าง Rule Class

```bash
php artisan make:rule ThaiPhoneNumber
php artisan make:rule StrongPassword
php artisan make:rule UniqueSlug
```

```php
<?php
// app/Rules/ThaiPhoneNumber.php

namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;

class ThaiPhoneNumber implements ValidationRule
{
    /**
     * Validate the attribute.
     * $value: ค่าที่ส่งมา validate
     * $fail: callback ที่เรียกเมื่อ fail
     */
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        // ลบ dash และ spaces
        $cleaned = preg_replace('/[\s\-]/', '', $value);

        // เบอร์ไทย: 0X-XXXX-XXXX (mobile) หรือ 0X-XXXX-XXXX (landline)
        $patterns = [
            '/^0[6-9]\d{8}$/',    // มือถือ: 06x, 07x, 08x, 09x
            '/^0[2-5]\d{7}$/',    // บ้าน Bangkok: 02x
            '/^0[3-9]\d{7,8}$/', // บ้านต่างจังหวัด
        ];

        $isValid = false;
        foreach ($patterns as $pattern) {
            if (preg_match($pattern, $cleaned)) {
                $isValid = true;
                break;
            }
        }

        if (!$isValid) {
            $fail('เบอร์โทรศัพท์ไม่ถูกต้อง กรุณากรอกเบอร์โทรไทย');
        }
    }
}
```

```php
<?php
// app/Rules/UniqueSlug.php

namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;
use Illuminate\Support\Facades\DB;

class UniqueSlug implements ValidationRule
{
    public function __construct(
        private string $table,
        private string $column = 'slug',
        private ?int $ignoreId = null,
    ) {}

    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        $query = DB::table($this->table)
            ->where($this->column, $value);

        if ($this->ignoreId) {
            $query->where('id', '!=', $this->ignoreId);
        }

        if ($query->exists()) {
            $fail("Slug นี้มีการใช้งานแล้ว กรุณาใช้ slug อื่น");
        }
    }
}
```

```php
<?php
// app/Rules/MaxWords.php

namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;

class MaxWords implements ValidationRule
{
    public function __construct(
        private int $maxWords
    ) {}

    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        $wordCount = str_word_count(strip_tags($value));
        
        if ($wordCount > $this->maxWords) {
            $fail(":attribute มีคำเกินกว่า {$this->maxWords} คำ (พบ {$wordCount} คำ)");
        }
    }
}
```

### 4.2 การใช้งาน Custom Rules

```php
<?php
use App\Rules\ThaiPhoneNumber;
use App\Rules\UniqueSlug;
use App\Rules\MaxWords;

$rules = [
    'phone'   => ['required', new ThaiPhoneNumber()],
    'slug'    => ['required', new UniqueSlug('posts', 'slug')],
    'excerpt' => ['required', new MaxWords(50)],
    
    // Update: ยกเว้น id ปัจจุบัน
    'slug'    => ['required', new UniqueSlug('posts', 'slug', $post->id)],
];
```

### 4.3 Closure Rules (สำหรับ one-time rules)

```php
<?php
use Illuminate\Validation\Validator;

$rules = [
    'username' => [
        'required',
        'string',
        'min:3',
        function (string $attribute, mixed $value, Closure $fail) {
            // ห้ามใช้ reserved words
            $reserved = ['admin', 'root', 'system', 'api', 'www'];
            
            if (in_array(strtolower($value), $reserved)) {
                $fail("ชื่อผู้ใช้ '{$value}' ไม่สามารถใช้งานได้");
            }
        },
    ],
];
```

### 4.4 Implicit Rules (ทำงานแม้ค่าเป็น empty)

```php
<?php
// app/Rules/NotBlacklisted.php

namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ImplicitRule;

class NotBlacklisted implements ImplicitRule
{
    private array $blacklist = ['spam@example.com', 'test@fake.com'];

    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        if (in_array(strtolower($value), $this->blacklist)) {
            $fail("อีเมลนี้ไม่สามารถใช้งานได้");
        }
    }
}
```

---

## 5. Conditional Validation

### 5.1 sometimes()

```php
<?php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($request->all(), [
    'name' => 'required',
    'age'  => 'nullable|integer',
]);

// เพิ่ม rule เฉพาะเมื่อมี field นั้น
$validator->sometimes('age', 'min:18', function ($input) {
    return $input->role === 'admin';
});

// เพิ่ม rule สำหรับ array items
$validator->sometimes('items.*.price', 'min:0', function ($input) {
    return $input->has_discount == false;
});
```

### 5.2 required_if / required_unless / required_with

```php
<?php
$rules = [
    'payment_method'    => 'required|in:credit_card,bank_transfer,cash',
    
    // required ถ้า payment_method == credit_card
    'card_number'       => 'required_if:payment_method,credit_card|nullable|string',
    'card_expiry'       => 'required_if:payment_method,credit_card|nullable|date_format:m/Y',
    'card_cvv'          => 'required_if:payment_method,credit_card|nullable|digits:3',
    
    // required ถ้า payment_method ไม่ใช่ cash
    'account_number'    => 'required_unless:payment_method,cash|nullable|string',
    
    // required ถ้ามี card_number
    'card_holder_name'  => 'required_with:card_number|nullable|string',
    
    // required ถ้ามีทั้ง a และ b
    'billing_address'   => 'required_with_all:card_number,card_expiry|nullable|string',
    
    // required ถ้าไม่มี company_name
    'personal_tax_id'   => 'required_without:company_tax_id|nullable',
];
```

### 5.3 Rule::when() และ Rule::unless()

```php
<?php
use Illuminate\Validation\Rule;

$rules = [
    'name' => 'required',
    'company' => [
        Rule::when($request->boolean('is_business'), [
            'required',
            'string',
            'min:2',
        ], 'nullable'), // else: nullable
    ],
    'tax_id' => [
        Rule::when(
            fn ($input) => $input->is_business,
            ['required', 'string', 'size:13'],
            'nullable'
        ),
    ],
];
```

### 5.4 Conditional Rules ใน Form Request

```php
<?php
// app/Http/Requests/UpdateProfileRequest.php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class UpdateProfileRequest extends FormRequest
{
    public function rules(): array
    {
        $userId = $this->user()->id;

        return [
            'name'     => 'required|string|max:255',
            'email'    => [
                'required',
                'email',
                Rule::unique('users', 'email')->ignore($userId),
            ],
            'password' => 'nullable|min:8|confirmed',

            // เฉพาะเมื่อต้องการเปลี่ยน password
            'current_password' => Rule::when(
                !empty($this->password),
                ['required', 'current_password'],
                'nullable'
            ),

            // Conditional based on role
            'department' => Rule::when(
                $this->user()->role === 'employee',
                ['required', 'exists:departments,id'],
                'nullable'
            ),
        ];
    }
}
```

---

## 6. Error Messages และการแสดงผล

### 6.1 Blade Templates

```php
<?php
// resources/views/auth/register.blade.php
?>

<form method="POST" action="{{ route('register') }}" enctype="multipart/form-data">
    @csrf

    <!-- แสดง all errors -->
    @if ($errors->any())
        <div class="alert alert-danger">
            <ul>
                @foreach ($errors->all() as $error)
                    <li>{{ $error }}</li>
                @endforeach
            </ul>
        </div>
    @endif

    <!-- Name field -->
    <div class="form-group">
        <label for="name">ชื่อ <span class="text-danger">*</span></label>
        <input 
            type="text" 
            id="name" 
            name="name" 
            value="{{ old('name') }}"
            class="form-control @error('name') is-invalid @enderror"
        >
        @error('name')
            <div class="invalid-feedback">{{ $message }}</div>
        @enderror
    </div>

    <!-- Email field -->
    <div class="form-group">
        <label for="email">อีเมล <span class="text-danger">*</span></label>
        <input 
            type="email" 
            id="email" 
            name="email" 
            value="{{ old('email') }}"
            class="form-control @error('email') is-invalid @enderror"
        >
        @error('email')
            <div class="invalid-feedback">{{ $message }}</div>
        @enderror
    </div>

    <button type="submit" class="btn btn-primary">ลงทะเบียน</button>
</form>
```

### 6.2 JSON API Responses

```php
<?php
// Form Request สำหรับ API (ส่ง JSON response)

namespace App\Http\Requests\Api;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Contracts\Validation\Validator;
use Illuminate\Http\Exceptions\HttpResponseException;

class StoreUserRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'name'  => 'required|string|max:255',
            'email' => 'required|email|unique:users',
        ];
    }

    /**
     * Override: คืน JSON response เมื่อ validation fail
     */
    protected function failedValidation(Validator $validator): void
    {
        throw new HttpResponseException(
            response()->json([
                'success' => false,
                'message' => 'ข้อมูลไม่ถูกต้อง กรุณาตรวจสอบและลองใหม่อีกครั้ง',
                'errors'  => $validator->errors(),
            ], 422)
        );
    }

    /**
     * Override: คืน JSON response เมื่อ authorize fail
     */
    protected function failedAuthorization(): void
    {
        throw new HttpResponseException(
            response()->json([
                'success' => false,
                'message' => 'ไม่มีสิทธิ์เข้าถึง',
            ], 403)
        );
    }
}
```

---

## 7. Workshop: User Registration Form Validation

### 7.1 Migration

```php
<?php
// database/migrations/create_users_table.php

Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('username')->unique();
    $table->string('email')->unique();
    $table->string('phone', 20)->nullable();
    $table->string('password');
    $table->date('birthday')->nullable();
    $table->enum('gender', ['male', 'female', 'other'])->nullable();
    $table->string('avatar')->nullable();
    $table->string('country_code', 2)->default('TH');
    $table->boolean('is_active')->default(true);
    $table->boolean('newsletter_subscribed')->default(false);
    $table->timestamp('email_verified_at')->nullable();
    $table->rememberToken();
    $table->timestamps();
});
```

### 7.2 Custom Rules

```php
<?php
// app/Rules/ThaiNationalId.php

namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;

class ThaiNationalId implements ValidationRule
{
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        // ลบ dash และ spaces
        $id = preg_replace('/[^0-9]/', '', $value);

        if (strlen($id) !== 13) {
            $fail('เลขบัตรประชาชนต้องมี 13 หลัก');
            return;
        }

        // Checksum algorithm สำหรับเลขบัตรประชาชนไทย
        $sum = 0;
        for ($i = 0; $i < 12; $i++) {
            $sum += (int) $id[$i] * (13 - $i);
        }

        $checkDigit = (11 - ($sum % 11)) % 10;

        if ($checkDigit !== (int) $id[12]) {
            $fail('เลขบัตรประชาชนไม่ถูกต้อง');
        }
    }
}
```

```php
<?php
// app/Rules/NotDisposableEmail.php

namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;

class NotDisposableEmail implements ValidationRule
{
    private array $disposableDomains = [
        'mailinator.com', 'guerrillamail.com', 'tempmail.com',
        '10minutemail.com', 'throwaway.email', 'sharklasers.com',
        'guerrillamailblock.com', 'grr.la', 'guerrillamail.info',
    ];

    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        $domain = substr(strrchr($value, "@"), 1);

        if (in_array(strtolower($domain), $this->disposableDomains)) {
            $fail('ไม่สามารถใช้อีเมลชั่วคราวได้');
        }
    }
}
```

### 7.3 Registration Form Request

```php
<?php
// app/Http/Requests/Auth/RegisterRequest.php

namespace App\Http\Requests\Auth;

use App\Rules\ThaiPhoneNumber;
use App\Rules\NotDisposableEmail;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;
use Illuminate\Validation\Rules\Password;

class RegisterRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true; // ทุกคนสมัครได้
    }

    public function rules(): array
    {
        return [
            // Personal Info
            'name' => [
                'required',
                'string',
                'min:2',
                'max:255',
                'regex:/^[\pL\s]+$/u', // ตัวอักษรและ spaces เท่านั้น (รองรับ Unicode/ไทย)
            ],

            'username' => [
                'required',
                'string',
                'min:3',
                'max:30',
                'regex:/^[a-z0-9_]+$/',  // lowercase, numbers, underscore เท่านั้น
                'unique:users,username',
                function (string $attribute, mixed $value, \Closure $fail) {
                    $reserved = ['admin', 'root', 'api', 'www', 'support', 'help', 'system'];
                    if (in_array(strtolower($value), $reserved)) {
                        $fail("ชื่อผู้ใช้ '{$value}' ไม่สามารถใช้งานได้");
                    }
                },
            ],

            'email' => [
                'required',
                'string',
                'email:rfc,dns',
                'max:255',
                'unique:users,email',
                new NotDisposableEmail(),
            ],

            'phone' => [
                'nullable',
                'string',
                new ThaiPhoneNumber(),
            ],

            // Password
            'password' => [
                'required',
                'confirmed',
                Password::min(8)
                    ->letters()
                    ->mixedCase()
                    ->numbers()
                    ->symbols()
                    ->uncompromised(),
            ],

            'password_confirmation' => 'required',

            // Personal Details
            'birthday' => [
                'nullable',
                'date',
                'before:' . now()->subYears(13)->format('Y-m-d'), // อายุอย่างน้อย 13 ปี
                'after:1900-01-01',
            ],

            'gender' => [
                'nullable',
                Rule::in(['male', 'female', 'other']),
            ],

            // Avatar
            'avatar' => [
                'nullable',
                'image',
                'mimes:jpeg,jpg,png,webp',
                'max:2048', // 2MB
                'dimensions:min_width=50,min_height=50,max_width=2000,max_height=2000',
            ],

            // Preferences
            'country_code' => [
                'required',
                'string',
                'size:2',
                'exists:countries,code',
            ],

            'newsletter_subscribed' => 'boolean',

            // Terms
            'terms' => 'required|accepted',
        ];
    }

    public function messages(): array
    {
        return [
            'name.required'              => 'กรุณากรอกชื่อ-นามสกุล',
            'name.regex'                 => 'ชื่อต้องเป็นตัวอักษรเท่านั้น',
            'username.required'          => 'กรุณากรอกชื่อผู้ใช้',
            'username.min'               => 'ชื่อผู้ใช้ต้องมีอย่างน้อย :min ตัวอักษร',
            'username.max'               => 'ชื่อผู้ใช้ต้องไม่เกิน :max ตัวอักษร',
            'username.regex'             => 'ชื่อผู้ใช้ต้องเป็นตัวพิมพ์เล็ก ตัวเลข หรือ _ เท่านั้น',
            'username.unique'            => 'ชื่อผู้ใช้นี้มีคนใช้งานแล้ว',
            'email.required'             => 'กรุณากรอกอีเมล',
            'email.email'                => 'รูปแบบอีเมลไม่ถูกต้อง',
            'email.unique'               => 'อีเมลนี้มีการสมัครแล้ว กรุณาใช้อีเมลอื่น',
            'password.required'          => 'กรุณากรอกรหัสผ่าน',
            'password.confirmed'         => 'รหัสผ่านไม่ตรงกัน',
            'birthday.before'            => 'ต้องมีอายุอย่างน้อย 13 ปีจึงจะสมัครได้',
            'avatar.image'               => 'ไฟล์ต้องเป็นรูปภาพ',
            'avatar.max'                 => 'รูปภาพต้องมีขนาดไม่เกิน 2MB',
            'avatar.dimensions'          => 'รูปภาพต้องมีขนาดระหว่าง 50x50 ถึง 2000x2000 pixels',
            'country_code.required'      => 'กรุณาเลือกประเทศ',
            'country_code.exists'        => 'รหัสประเทศไม่ถูกต้อง',
            'terms.accepted'             => 'กรุณายอมรับเงื่อนไขการใช้งานก่อนสมัคร',
        ];
    }

    public function attributes(): array
    {
        return [
            'name'                   => 'ชื่อ-นามสกุล',
            'username'               => 'ชื่อผู้ใช้',
            'email'                  => 'อีเมล',
            'phone'                  => 'เบอร์โทร',
            'password'               => 'รหัสผ่าน',
            'password_confirmation'  => 'ยืนยันรหัสผ่าน',
            'birthday'               => 'วันเกิด',
            'gender'                 => 'เพศ',
            'avatar'                 => 'รูปโปรไฟล์',
            'country_code'           => 'ประเทศ',
            'newsletter_subscribed'  => 'รับข่าวสาร',
            'terms'                  => 'เงื่อนไขการใช้งาน',
        ];
    }

    protected function prepareForValidation(): void
    {
        $this->merge([
            'email'    => strtolower(trim($this->email ?? '')),
            'username' => strtolower(trim($this->username ?? '')),
            'name'     => trim($this->name ?? ''),
        ]);
    }
}
```

### 7.4 Registration Controller

```php
<?php
// app/Http/Controllers/Auth/RegisterController.php

namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use App\Http\Requests\Auth\RegisterRequest;
use App\Models\User;
use Illuminate\Auth\Events\Registered;
use Illuminate\Http\RedirectResponse;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\Storage;

class RegisterController extends Controller
{
    public function showForm()
    {
        return view('auth.register');
    }

    public function register(RegisterRequest $request): RedirectResponse
    {
        $validated = $request->validated();

        // Handle avatar upload
        $avatarPath = null;
        if ($request->hasFile('avatar')) {
            $avatarPath = $request->file('avatar')
                ->store('avatars', 'public');
        }

        // Create user
        $user = User::create([
            'name'                   => $validated['name'],
            'username'               => $validated['username'],
            'email'                  => $validated['email'],
            'phone'                  => $validated['phone'] ?? null,
            'password'               => Hash::make($validated['password']),
            'birthday'               => $validated['birthday'] ?? null,
            'gender'                 => $validated['gender'] ?? null,
            'avatar'                 => $avatarPath,
            'country_code'           => $validated['country_code'],
            'newsletter_subscribed'  => $validated['newsletter_subscribed'] ?? false,
        ]);

        // Trigger registered event (email verification)
        event(new Registered($user));

        // Login
        Auth::login($user);

        return redirect()->route('dashboard')
            ->with('success', 'ยินดีต้อนรับ! สมัครสมาชิกสำเร็จ กรุณายืนยันอีเมลของคุณ');
    }
}
```

### 7.5 Update Profile Request

```php
<?php
// app/Http/Requests/UpdateProfileRequest.php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;
use Illuminate\Validation\Rules\Password;

class UpdateProfileRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        $userId = $this->user()->id;

        return [
            'name'  => 'required|string|max:255',
            'email' => [
                'required',
                'email',
                Rule::unique('users', 'email')->ignore($userId),
            ],

            // Password change (optional)
            'current_password' => [
                Rule::when(
                    !empty($this->new_password),
                    ['required', 'current_password']
                ),
            ],
            'new_password' => [
                'nullable',
                'confirmed',
                Password::min(8)->letters()->mixedCase()->numbers(),
            ],

            'phone'  => 'nullable|string',
            'bio'    => 'nullable|string|max:500',
            'avatar' => 'nullable|image|max:2048',
        ];
    }

    public function messages(): array
    {
        return [
            'current_password.required'       => 'กรุณากรอกรหัสผ่านปัจจุบัน',
            'current_password.current_password' => 'รหัสผ่านปัจจุบันไม่ถูกต้อง',
            'new_password.confirmed'           => 'รหัสผ่านใหม่ไม่ตรงกัน',
        ];
    }
}
```

---

## Quiz พร้อมเฉลย

### คำถามที่ 1
ความแตกต่างระหว่าง `validate()` ใน Controller กับ Form Request คืออะไร?

**เฉลย:**
- `$request->validate()` ใน Controller: เขียนรวมกับ business logic ใน Controller, เหมาะสำหรับ validation ง่ายๆ
- Form Request: แยก validation ออกมาเป็น class เดียวกัน, reusable, test ได้ง่ายกว่า, เหมาะสำหรับ validation ที่ซับซ้อน
- Form Request ยังมี `authorize()` method สำหรับ authorization ด้วย

### คำถามที่ 2
`required_if` และ `required_unless` ต่างกันอย่างไร?

**เฉลย:**
```php
// required_if: required เมื่อ field อื่น == ค่าที่กำหนด
'credit_card_number' => 'required_if:payment,credit_card'
// required ถ้า payment == 'credit_card'

// required_unless: required เมื่อ field อื่น != ค่าที่กำหนด
'account_number' => 'required_unless:payment,cash'
// required ถ้า payment ไม่ใช่ 'cash'
```

### คำถามที่ 3
เขียน Custom Rule ที่ validate ว่า email domain ต้องเป็น domain ที่อนุญาต

**เฉลย:**
```php
// app/Rules/AllowedEmailDomain.php
class AllowedEmailDomain implements ValidationRule
{
    public function __construct(
        private array $allowedDomains = ['company.com', 'partner.com']
    ) {}

    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        $domain = substr(strrchr($value, "@"), 1);
        
        if (!in_array(strtolower($domain), $this->allowedDomains)) {
            $fail("กรุณาใช้อีเมลจาก domain ที่อนุญาต: " . implode(', ', $this->allowedDomains));
        }
    }
}

// การใช้งาน:
'email' => ['required', 'email', new AllowedEmailDomain(['company.com'])],
```

### คำถามที่ 4
อธิบาย `$request->validated()` vs `$request->all()`

**เฉลย:**
- `$request->all()`: คืนข้อมูลทั้งหมดที่ส่งมา (อาจมีข้อมูลที่ไม่ต้องการ)
- `$request->validated()`: คืนเฉพาะข้อมูลที่ผ่าน validation rules เท่านั้น
- ควรใช้ `$request->validated()` เสมอ เพื่อความปลอดภัยและ mass assignment

### คำถามที่ 5
เมื่อ Validation ล้มเหลวใน API (JSON request) จะเกิดอะไรขึ้น?

**เฉลย:**
Laravel จะ return JSON response 422 อัตโนมัติ:
```json
{
    "message": "The email field must be a valid email address.",
    "errors": {
        "email": [
            "The email field must be a valid email address."
        ]
    }
}
```
หรือกำหนด custom response ผ่าน `failedValidation()` method

---

## แบบฝึกหัด

### Exercise 1: Product Form Validation
สร้าง `StoreProductRequest` ที่ validate:
- name: required, unique ใน products table
- price: required, numeric, min 0
- compare_price: nullable, ต้องมากกว่า price
- category_id: required, exists ใน categories
- images: array, แต่ละรูปต้องเป็น image ไม่เกิน 5MB

### Exercise 2: Import CSV Validation
สร้าง Form Request สำหรับ import products จาก CSV:
- file: required, mimes:csv,txt, max:10240
- overwrite: boolean
- สร้าง Custom Rule ที่ validate CSV format และ headers

### Exercise 3: Conditional Payment Form
สร้าง Form Request สำหรับ payment ที่มี conditional rules:
- ถ้า payment_method = credit_card: card_number, expiry, cvv required
- ถ้า payment_method = bank_transfer: bank_name, account_number required
- ถ้า payment_method = promptpay: promptpay_id required

---

## สรุป

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Inline Validation | `$request->validate()` |
| Form Request | `authorize()`, `rules()`, `messages()`, `attributes()` |
| Validation Rules | required, string, numeric, date, database, file, array |
| Custom Rules | Rule class, Closure, Implicit |
| Conditional | `sometimes()`, `required_if`, `Rule::when()` |
| Error Display | `@error`, `$errors`, JSON responses |

---

## ลิงก์ไป Part ถัดไป

➡️ [Part 034: Laravel Authentication](part-034-laravel-authentication.md)

ใน Part ถัดไปเราจะเรียนเรื่อง Authentication ซึ่งรวม Breeze, Sanctum, Guards, Policies และ Email Verification
