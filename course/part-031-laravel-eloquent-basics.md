# Part 031: Laravel Eloquent ORM - พื้นฐาน

## ระดับ: Intermediate
## เวลาที่ใช้: 4-5 ชั่วโมง

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้างและจัดการ Eloquent Model ได้อย่างถูกต้อง
- ทำ CRUD operations ด้วย Eloquent ORM
- ใช้ Query Builder ขั้นสูง
- สร้าง Scopes สำหรับ reusable queries
- ใช้ Mutators, Accessors, และ Casts
- สร้างระบบ Product Catalog ที่ใช้งานได้จริง

---

## 1. Eloquent ORM คืออะไร?

Eloquent ORM (Object-Relational Mapping) คือ Active Record implementation ของ Laravel ที่ให้คุณทำงานกับฐานข้อมูลโดยใช้ PHP objects แทน SQL ดิบ

### ข้อดีของ Eloquent:
- เขียนโค้ดน้อยกว่า SQL ดิบ
- ป้องกัน SQL Injection โดยอัตโนมัติ
- รองรับ relationships ระหว่าง tables
- มี built-in validation และ casting
- ง่ายต่อการ unit test

---

## 2. การสร้าง Model

### 2.1 คำสั่งพื้นฐาน

```bash
# สร้าง Model อย่างเดียว
php artisan make:model Product

# สร้าง Model พร้อม Migration
php artisan make:model Product -m

# สร้าง Model พร้อม Migration, Factory, Seeder
php artisan make:model Product -mfs

# สร้าง Model พร้อมทุกอย่าง (Migration, Factory, Seeder, Controller, Request)
php artisan make:model Product --all

# ดู options ทั้งหมด
php artisan make:model --help
```

### 2.2 โครงสร้าง Model พื้นฐาน

```php
<?php
// app/Models/Product.php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Product extends Model
{
    use HasFactory, SoftDeletes;

    /**
     * ชื่อ table (ถ้าต่างจาก convention)
     * Convention: ชื่อ class เป็น singular, table เป็น plural
     * Product -> products (อัตโนมัติ)
     */
    protected $table = 'products';

    /**
     * Primary key (default: id)
     */
    protected $primaryKey = 'id';

    /**
     * Auto-increment (default: true)
     */
    public $incrementing = true;

    /**
     * ประเภทของ primary key (default: int)
     */
    protected $keyType = 'int';

    /**
     * Timestamps (default: true)
     * จะจัดการ created_at และ updated_at อัตโนมัติ
     */
    public $timestamps = true;

    /**
     * Fields ที่อนุญาตให้ mass assignment
     * ควรระบุ $fillable หรือ $guarded อย่างใดอย่างหนึ่ง
     */
    protected $fillable = [
        'name',
        'description',
        'price',
        'stock_quantity',
        'category_id',
        'sku',
        'is_active',
    ];

    /**
     * Fields ที่ไม่อนุญาตให้ mass assignment
     * ใช้ $guarded แทน $fillable ก็ได้
     */
    // protected $guarded = ['id', 'created_at', 'updated_at'];

    /**
     * Default attribute values
     */
    protected $attributes = [
        'is_active' => true,
        'stock_quantity' => 0,
    ];
}
```

### 2.3 สร้าง Migration

```php
<?php
// database/migrations/2024_01_01_000000_create_products_table.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('products', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('slug')->unique();
            $table->text('description')->nullable();
            $table->decimal('price', 10, 2);
            $table->decimal('compare_price', 10, 2)->nullable();
            $table->integer('stock_quantity')->default(0);
            $table->string('sku')->unique()->nullable();
            $table->string('image_url')->nullable();
            $table->foreignId('category_id')->constrained()->onDelete('cascade');
            $table->boolean('is_active')->default(true);
            $table->boolean('is_featured')->default(false);
            $table->timestamps();
            $table->softDeletes(); // deleted_at column สำหรับ soft delete
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('products');
    }
};
```

```bash
# รัน migration
php artisan migrate

# ย้อน migration ล่าสุด
php artisan migrate:rollback

# ย้อนทั้งหมดแล้วรันใหม่
php artisan migrate:fresh

# ดู status ของ migrations
php artisan migrate:status
```

---

## 3. CRUD ด้วย Eloquent

### 3.1 Create (สร้างข้อมูล)

```php
<?php
// วิธีที่ 1: create() - Mass Assignment (ต้องมี $fillable หรือ $guarded)
$product = Product::create([
    'name' => 'iPhone 15 Pro',
    'slug' => 'iphone-15-pro',
    'description' => 'Apple iPhone 15 Pro with A17 Pro chip',
    'price' => 45900.00,
    'stock_quantity' => 50,
    'sku' => 'APPLE-IP15P-001',
    'category_id' => 1,
    'is_active' => true,
]);

// วิธีที่ 2: new + save()
$product = new Product();
$product->name = 'Samsung Galaxy S24';
$product->slug = 'samsung-galaxy-s24';
$product->price = 35900.00;
$product->category_id = 1;
$product->save();

// วิธีที่ 3: firstOrCreate - สร้างถ้าไม่มี, หาถ้ามี
$product = Product::firstOrCreate(
    ['sku' => 'APPLE-IP15P-001'], // conditions สำหรับค้นหา
    [                              // values สำหรับสร้างถ้าไม่มี
        'name' => 'iPhone 15 Pro',
        'price' => 45900.00,
        'category_id' => 1,
    ]
);

// วิธีที่ 4: updateOrCreate - สร้างหรืออัพเดต
$product = Product::updateOrCreate(
    ['sku' => 'APPLE-IP15P-001'], // conditions
    [
        'name' => 'iPhone 15 Pro',
        'price' => 45900.00,
        'stock_quantity' => 100,
    ]
);
```

### 3.2 Read (อ่านข้อมูล)

```php
<?php
// ดึงทุก records
$products = Product::all();

// ดึงด้วย primary key
$product = Product::find(1);
$product = Product::findOrFail(1); // ถ้าไม่เจอ throw ModelNotFoundException

// ดึงหลาย records ด้วย array ของ IDs
$products = Product::find([1, 2, 3]);
$products = Product::findMany([1, 2, 3]);

// ดึงด้วยเงื่อนไข
$products = Product::where('is_active', true)->get();
$product = Product::where('slug', 'iphone-15-pro')->first();
$product = Product::where('slug', 'iphone-15-pro')->firstOrFail();

// ดึงพร้อมเงื่อนไขหลายอย่าง
$products = Product::where('is_active', true)
    ->where('price', '>', 10000)
    ->where('stock_quantity', '>', 0)
    ->get();

// ดึงแบบ OR
$products = Product::where('name', 'like', '%iPhone%')
    ->orWhere('name', 'like', '%Samsung%')
    ->get();

// ดึงด้วย whereIn
$products = Product::whereIn('category_id', [1, 2, 3])->get();

// ดึงด้วย whereBetween
$products = Product::whereBetween('price', [10000, 50000])->get();

// ดึงด้วย whereNull / whereNotNull
$products = Product::whereNull('deleted_at')->get();
$products = Product::whereNotNull('image_url')->get();

// นับจำนวน
$count = Product::count();
$activeCount = Product::where('is_active', true)->count();

// Aggregate functions
$maxPrice = Product::max('price');
$minPrice = Product::min('price');
$avgPrice = Product::avg('price');
$totalStock = Product::sum('stock_quantity');

// Pagination
$products = Product::paginate(15); // 15 items per page
$products = Product::simplePaginate(15); // ไม่มีข้อมูล total pages

// Cursor Pagination (efficient สำหรับ large datasets)
$products = Product::cursorPaginate(15);

// Chunk สำหรับ large datasets
Product::chunk(100, function ($products) {
    foreach ($products as $product) {
        // ประมวลผลทีละ 100 records
    }
});

// Lazy collection
foreach (Product::lazy() as $product) {
    // ดึงข้อมูลทีละส่วน ประหยัด memory
}
```

### 3.3 Update (อัพเดตข้อมูล)

```php
<?php
// วิธีที่ 1: ดึงแล้วอัพเดต
$product = Product::find(1);
$product->price = 42000.00;
$product->stock_quantity = 75;
$product->save();

// วิธีที่ 2: fill() + save()
$product = Product::find(1);
$product->fill([
    'price' => 42000.00,
    'stock_quantity' => 75,
]);
$product->save();

// วิธีที่ 3: update() - Mass Update
Product::where('category_id', 1)->update([
    'is_featured' => true,
]);

// วิธีที่ 4: updateOrCreate
Product::updateOrCreate(
    ['sku' => 'APPLE-IP15P-001'],
    ['price' => 42000.00, 'stock_quantity' => 75]
);

// Increment / Decrement
$product = Product::find(1);
$product->increment('stock_quantity'); // เพิ่ม 1
$product->increment('stock_quantity', 5); // เพิ่ม 5
$product->decrement('stock_quantity', 2); // ลด 2

// Increment แบบ query
Product::where('is_featured', true)->increment('view_count');
```

### 3.4 Delete (ลบข้อมูล)

```php
<?php
// Hard Delete - ลบถาวร
$product = Product::find(1);
$product->delete();

// Delete แบบ query
Product::where('is_active', false)->delete();
Product::whereIn('id', [1, 2, 3])->delete();

// Soft Delete - ต้องใช้ SoftDeletes trait
$product = Product::find(1);
$product->delete(); // จะ set deleted_at เป็น timestamp ปัจจุบัน

// ดึงข้อมูลที่ถูก soft delete ด้วย
$products = Product::withTrashed()->get();

// ดึงเฉพาะที่ถูก soft delete
$deletedProducts = Product::onlyTrashed()->get();

// Restore soft deleted record
$product = Product::withTrashed()->find(1);
$product->restore();

// Force delete - ลบถาวรแม้จะมี soft delete
$product->forceDelete();
```

---

## 4. Query Builder

### 4.1 Basic Queries

```php
<?php
use Illuminate\Support\Facades\DB;

// Raw Query (ควรหลีกเลี่ยง)
$products = DB::select('SELECT * FROM products WHERE is_active = ?', [true]);

// Query Builder
$products = DB::table('products')
    ->where('is_active', true)
    ->get();

// Eloquent Query Builder (ดีกว่า)
$products = Product::query()
    ->where('is_active', true)
    ->orderBy('created_at', 'desc')
    ->get();
```

### 4.2 Advanced Queries

```php
<?php
// Select specific columns
$products = Product::select('id', 'name', 'price')
    ->where('is_active', true)
    ->get();

// selectRaw - ใช้ SQL expressions
$products = Product::selectRaw('id, name, price, price * 0.07 as vat')
    ->where('is_active', true)
    ->get();

// Join
$products = Product::join('categories', 'products.category_id', '=', 'categories.id')
    ->select('products.*', 'categories.name as category_name')
    ->where('products.is_active', true)
    ->get();

// Left Join
$products = Product::leftJoin('order_items', 'products.id', '=', 'order_items.product_id')
    ->select('products.*', DB::raw('COUNT(order_items.id) as order_count'))
    ->groupBy('products.id')
    ->get();

// Subquery
$popularProducts = Product::whereIn('id', function ($query) {
    $query->select('product_id')
        ->from('order_items')
        ->groupBy('product_id')
        ->havingRaw('COUNT(*) > 10');
})->get();

// whereExists
$productsWithOrders = Product::whereExists(function ($query) {
    $query->select(DB::raw(1))
        ->from('order_items')
        ->whereColumn('order_items.product_id', 'products.id');
})->get();

// Ordering
$products = Product::orderBy('price', 'asc')
    ->orderBy('name', 'asc')
    ->get();

$products = Product::orderByDesc('created_at')->get();
$products = Product::latest()->get(); // orderBy('created_at', 'desc')
$products = Product::oldest()->get(); // orderBy('created_at', 'asc')

// Limit & Offset
$products = Product::limit(10)->offset(20)->get();
$products = Product::skip(20)->take(10)->get(); // เหมือนกัน

// distinct
$categories = Product::distinct()->pluck('category_id');

// Group By
$categoryStats = Product::selectRaw('category_id, COUNT(*) as count, AVG(price) as avg_price')
    ->groupBy('category_id')
    ->having('count', '>', 5)
    ->get();

// Group By หลายคอลัมน์
$stats = Product::selectRaw('category_id, is_active, COUNT(*) as count')
    ->groupBy('category_id', 'is_active')
    ->get();
```

### 4.3 Useful Query Methods

```php
<?php
// exists() - เช็คว่ามีข้อมูลหรือไม่
$exists = Product::where('sku', 'APPLE-IP15P-001')->exists();
$notExists = Product::where('sku', 'INVALID')->doesntExist();

// value() - ดึง single value
$price = Product::where('id', 1)->value('price');

// pluck() - ดึง array ของ column เดียว
$productNames = Product::pluck('name');
$productPrices = Product::pluck('price', 'id'); // key => value

// toArray() - แปลงเป็น array
$productsArray = Product::all()->toArray();

// toJson() - แปลงเป็น JSON
$productsJson = Product::all()->toJson();

// lists() vs pluck()
$names = Product::all()->pluck('name')->toArray();

// map() - transform collection
$formattedProducts = Product::all()->map(function ($product) {
    return [
        'id' => $product->id,
        'name' => $product->name,
        'formatted_price' => '฿' . number_format($product->price, 2),
    ];
});

// filter() - filter collection
$expensiveProducts = Product::all()->filter(function ($product) {
    return $product->price > 10000;
});

// sortBy() - sort collection
$sortedByPrice = Product::all()->sortBy('price');
$sortedByPriceDesc = Product::all()->sortByDesc('price');

// groupBy() - group collection
$groupedByCategory = Product::all()->groupBy('category_id');

// unique() - ลบ duplicates
$uniqueCategories = Product::all()->unique('category_id');

// first() และ last()
$cheapest = Product::all()->sortBy('price')->first();
$expensive = Product::all()->sortByDesc('price')->first();
```

---

## 5. Scopes

Scopes ช่วยให้คุณ encapsulate query constraints ที่ใช้บ่อยๆ

### 5.1 Local Scopes

```php
<?php
// app/Models/Product.php

class Product extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = ['name', 'price', 'is_active', 'is_featured', 'stock_quantity', 'category_id'];

    /**
     * Local Scope: เฉพาะสินค้าที่ active
     * การใช้: Product::active()->get()
     */
    public function scopeActive($query)
    {
        return $query->where('is_active', true);
    }

    /**
     * Local Scope: เฉพาะสินค้าที่ featured
     */
    public function scopeFeatured($query)
    {
        return $query->where('is_featured', true);
    }

    /**
     * Local Scope: สินค้าที่มี stock
     */
    public function scopeInStock($query)
    {
        return $query->where('stock_quantity', '>', 0);
    }

    /**
     * Local Scope: กรองตาม category
     * การใช้: Product::ofCategory(1)->get()
     */
    public function scopeOfCategory($query, int $categoryId)
    {
        return $query->where('category_id', $categoryId);
    }

    /**
     * Local Scope: ราคาในช่วงที่กำหนด
     * การใช้: Product::priceRange(1000, 5000)->get()
     */
    public function scopePriceRange($query, float $min, float $max)
    {
        return $query->whereBetween('price', [$min, $max]);
    }

    /**
     * Local Scope: ค้นหาด้วยชื่อ
     */
    public function scopeSearch($query, string $term)
    {
        return $query->where(function ($q) use ($term) {
            $q->where('name', 'like', "%{$term}%")
              ->orWhere('description', 'like', "%{$term}%")
              ->orWhere('sku', 'like', "%{$term}%");
        });
    }

    /**
     * Local Scope: เรียงตาม field ที่กำหนด
     */
    public function scopeSortBy($query, string $field = 'created_at', string $direction = 'desc')
    {
        $allowedFields = ['name', 'price', 'created_at', 'stock_quantity'];
        
        if (!in_array($field, $allowedFields)) {
            $field = 'created_at';
        }

        return $query->orderBy($field, $direction);
    }
}
```

### 5.2 การใช้งาน Local Scopes

```php
<?php
// ใช้ scope เดี่ยว
$activeProducts = Product::active()->get();
$featuredProducts = Product::featured()->get();
$inStockProducts = Product::inStock()->get();

// Chain scopes
$activeInStockFeatured = Product::active()
    ->inStock()
    ->featured()
    ->get();

// Scope พร้อม parameters
$electronicProducts = Product::ofCategory(1)->get();
$midRangeProducts = Product::priceRange(10000, 30000)->get();
$searchResults = Product::search('iPhone')->get();

// Chain scopes กับ query methods ปกติ
$results = Product::active()
    ->inStock()
    ->ofCategory(1)
    ->priceRange(5000, 50000)
    ->search('Pro')
    ->sortBy('price', 'asc')
    ->paginate(20);
```

### 5.3 Global Scopes

```php
<?php
// app/Scopes/ActiveScope.php

namespace App\Scopes;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Scope;

class ActiveScope implements Scope
{
    /**
     * Apply the scope to a given Eloquent query builder.
     */
    public function apply(Builder $builder, Model $model): void
    {
        $builder->where('is_active', true);
    }
}
```

```php
<?php
// app/Models/Product.php

use App\Scopes\ActiveScope;

class Product extends Model
{
    /**
     * Boot method ที่รันอัตโนมัติเมื่อ Model ถูกใช้งาน
     */
    protected static function booted(): void
    {
        // เพิ่ม global scope
        static::addGlobalScope(new ActiveScope());

        // หรือใช้ anonymous scope
        static::addGlobalScope('active', function (Builder $builder) {
            $builder->where('is_active', true);
        });
    }
}
```

```php
<?php
// การใช้ Global Scope
$products = Product::all(); // จะ filter เฉพาะ active products อัตโนมัติ

// ข้าม global scope
$allProducts = Product::withoutGlobalScope('active')->get();
$allProducts = Product::withoutGlobalScopes()->get(); // ข้ามทุก scopes
$allProducts = Product::withoutGlobalScope(ActiveScope::class)->get();
```

---

## 6. Mutators & Accessors

### 6.1 Accessors (แปลงค่าเมื่ออ่าน)

```php
<?php
// app/Models/Product.php

use Illuminate\Database\Eloquent\Casts\Attribute;

class Product extends Model
{
    /**
     * Accessor แบบใหม่ (Laravel 9+)
     * ชื่อ method = ชื่อ attribute (camelCase)
     * จะ access ได้จาก $product->formatted_price
     */
    protected function formattedPrice(): Attribute
    {
        return Attribute::make(
            get: fn ($value, $attributes) => '฿' . number_format($attributes['price'], 2),
        );
    }

    /**
     * Accessor: ราคาหลังหักส่วนลด
     */
    protected function discountedPrice(): Attribute
    {
        return Attribute::make(
            get: function ($value, $attributes) {
                if ($attributes['compare_price'] && $attributes['compare_price'] > $attributes['price']) {
                    $discount = (($attributes['compare_price'] - $attributes['price']) / $attributes['compare_price']) * 100;
                    return round($discount);
                }
                return 0;
            }
        );
    }

    /**
     * Accessor: ชื่อสินค้าตัวพิมพ์ใหญ่
     */
    protected function name(): Attribute
    {
        return Attribute::make(
            get: fn ($value) => ucwords(strtolower($value)),
            set: fn ($value) => strtolower($value), // Mutator ด้วย
        );
    }

    /**
     * Accessor: สถานะ stock
     */
    protected function stockStatus(): Attribute
    {
        return Attribute::make(
            get: function ($value, $attributes) {
                $quantity = $attributes['stock_quantity'];
                
                if ($quantity === 0) {
                    return 'out_of_stock';
                } elseif ($quantity <= 5) {
                    return 'low_stock';
                } else {
                    return 'in_stock';
                }
            }
        );
    }
}
```

### 6.2 Mutators (แปลงค่าเมื่อบันทึก)

```php
<?php
// app/Models/Product.php

class Product extends Model
{
    /**
     * Mutator + Accessor ในที่เดียว
     */
    protected function slug(): Attribute
    {
        return Attribute::make(
            get: fn ($value) => $value,
            set: fn ($value) => Str::slug($value), // แปลงเป็น slug เมื่อ set
        );
    }

    /**
     * แปลง price ให้ validate ก่อนบันทึก
     */
    protected function price(): Attribute
    {
        return Attribute::make(
            get: fn ($value) => (float) $value,
            set: function ($value) {
                if ($value < 0) {
                    throw new \InvalidArgumentException('Price cannot be negative');
                }
                return (float) $value;
            }
        );
    }

    /**
     * Mutator: แปลง JSON string เป็น array
     */
    protected function specifications(): Attribute
    {
        return Attribute::make(
            get: fn ($value) => json_decode($value, true),
            set: fn ($value) => json_encode($value),
        );
    }
}
```

### 6.3 การใช้งาน

```php
<?php
$product = Product::find(1);

// Accessor
echo $product->formatted_price;    // ฿45,900.00
echo $product->discounted_price;   // 10 (เปอร์เซ็นต์)
echo $product->stock_status;       // in_stock
echo $product->name;               // Iphone 15 Pro

// Mutator
$product->name = 'IPHONE 15 PRO';  // จะถูกแปลงเป็น 'iphone 15 pro'
$product->slug = 'iPhone 15 Pro';  // จะถูกแปลงเป็น 'iphone-15-pro'
$product->save();

// เพิ่ม computed attributes ลงใน JSON/Array output
// ต้องเพิ่มใน $appends
```

### 6.4 $appends - เพิ่ม Computed Attributes

```php
<?php
// app/Models/Product.php

class Product extends Model
{
    protected $appends = [
        'formatted_price',
        'discounted_price', 
        'stock_status',
    ];

    // ตอนนี้ $product->toArray() และ $product->toJson()
    // จะรวม formatted_price, discounted_price, stock_status ด้วย
}
```

---

## 7. Casts

Casts ช่วยแปลงค่าจากฐานข้อมูลเป็นประเภทที่ต้องการอัตโนมัติ

### 7.1 Built-in Casts

```php
<?php
// app/Models/Product.php

class Product extends Model
{
    /**
     * Attribute casts
     */
    protected $casts = [
        // Primitive types
        'id'                => 'integer',
        'price'             => 'float',
        'stock_quantity'    => 'integer',
        'is_active'         => 'boolean',
        'is_featured'       => 'boolean',
        
        // Date/Time
        'published_at'      => 'datetime',
        'expired_at'        => 'date',
        'sale_ends_at'      => 'immutable_datetime', // ไม่เปลี่ยนแปลงได้
        
        // JSON
        'specifications'    => 'array',    // JSON -> PHP array
        'metadata'          => 'object',   // JSON -> stdClass object
        'settings'          => 'collection', // JSON -> Laravel Collection
        
        // Hashing (Laravel 10+)
        'password'          => 'hashed',   // auto hash ด้วย bcrypt
        
        // Encrypted
        'secret_key'        => 'encrypted', // encrypt/decrypt อัตโนมัติ
        'payment_data'      => 'encrypted:array',
    ];
}
```

### 7.2 Custom Cast Class

```php
<?php
// app/Casts/MoneyCast.php

namespace App\Casts;

use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
use Illuminate\Database\Eloquent\Model;

class MoneyCast implements CastsAttributes
{
    public function __construct(
        protected string $currency = 'THB'
    ) {}

    /**
     * Cast the given value (อ่านจาก DB)
     */
    public function get(Model $model, string $key, mixed $value, array $attributes): mixed
    {
        return [
            'amount' => (float) $value,
            'currency' => $this->currency,
            'formatted' => $this->format((float) $value),
        ];
    }

    /**
     * Prepare the given value for storage (บันทึกลง DB)
     */
    public function set(Model $model, string $key, mixed $value, array $attributes): mixed
    {
        if (is_array($value)) {
            return $value['amount'];
        }
        
        return $value;
    }

    private function format(float $amount): string
    {
        return match($this->currency) {
            'THB' => '฿' . number_format($amount, 2),
            'USD' => '$' . number_format($amount, 2),
            default => number_format($amount, 2) . ' ' . $this->currency,
        };
    }
}
```

```php
<?php
// app/Models/Product.php

use App\Casts\MoneyCast;

class Product extends Model
{
    protected $casts = [
        'price'         => MoneyCast::class,
        'compare_price' => MoneyCast::class . ':USD',
    ];
}
```

### 7.3 Enum Casts (PHP 8.1+)

```php
<?php
// app/Enums/ProductStatus.php

namespace App\Enums;

enum ProductStatus: string
{
    case Active = 'active';
    case Inactive = 'inactive';
    case Draft = 'draft';
    case Discontinued = 'discontinued';

    public function label(): string
    {
        return match($this) {
            ProductStatus::Active => 'ใช้งาน',
            ProductStatus::Inactive => 'ไม่ใช้งาน',
            ProductStatus::Draft => 'แบบร่าง',
            ProductStatus::Discontinued => 'ยกเลิก',
        };
    }

    public function color(): string
    {
        return match($this) {
            ProductStatus::Active => 'green',
            ProductStatus::Inactive => 'gray',
            ProductStatus::Draft => 'yellow',
            ProductStatus::Discontinued => 'red',
        };
    }
}
```

```php
<?php
// app/Models/Product.php

use App\Enums\ProductStatus;

class Product extends Model
{
    protected $casts = [
        'status' => ProductStatus::class,
    ];
}
```

```php
<?php
// การใช้งาน Enum Cast
$product = Product::find(1);

// อ่านค่า
echo $product->status->value;   // 'active'
echo $product->status->label(); // 'ใช้งาน'
echo $product->status->color(); // 'green'

// เปรียบเทียบ
if ($product->status === ProductStatus::Active) {
    // ...
}

// บันทึก
$product->status = ProductStatus::Inactive;
$product->save();

// Query ด้วย Enum
$activeProducts = Product::where('status', ProductStatus::Active)->get();
$activeProducts = Product::where('status', 'active')->get(); // ก็ได้
```

---

## 8. Events & Observers

### 8.1 Model Events

```php
<?php
// app/Models/Product.php

use Illuminate\Support\Str;

class Product extends Model
{
    protected static function booted(): void
    {
        // ก่อน creating
        static::creating(function (Product $product) {
            if (empty($product->slug)) {
                $product->slug = Str::slug($product->name);
            }
            if (empty($product->sku)) {
                $product->sku = 'PROD-' . strtoupper(Str::random(8));
            }
        });

        // หลัง created
        static::created(function (Product $product) {
            // ส่ง notification
            // Cache::tags('products')->flush();
        });

        // ก่อน updating
        static::updating(function (Product $product) {
            if ($product->isDirty('name')) {
                $product->slug = Str::slug($product->name);
            }
        });

        // ก่อน deleting
        static::deleting(function (Product $product) {
            // ลบรูปภาพก่อนลบ record
            if ($product->image_url) {
                Storage::delete($product->image_url);
            }
        });
    }
}
```

### 8.2 Observer Pattern

```php
<?php
// app/Observers/ProductObserver.php

namespace App\Observers;

use App\Models\Product;
use Illuminate\Support\Str;

class ProductObserver
{
    public function creating(Product $product): void
    {
        if (empty($product->slug)) {
            $product->slug = Str::slug($product->name);
        }
    }

    public function created(Product $product): void
    {
        \Log::info("Product created: {$product->name} (ID: {$product->id})");
    }

    public function updating(Product $product): void
    {
        if ($product->isDirty('name')) {
            $product->slug = Str::slug($product->name);
        }
    }

    public function updated(Product $product): void
    {
        // Clear cache
        \Cache::forget("product.{$product->id}");
    }

    public function deleted(Product $product): void
    {
        \Log::info("Product deleted: {$product->name} (ID: {$product->id})");
    }

    public function restored(Product $product): void
    {
        \Log::info("Product restored: {$product->name} (ID: {$product->id})");
    }

    public function forceDeleted(Product $product): void
    {
        \Log::info("Product force deleted: {$product->name} (ID: {$product->id})");
    }
}
```

```php
<?php
// app/Providers/AppServiceProvider.php

use App\Models\Product;
use App\Observers\ProductObserver;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        Product::observe(ProductObserver::class);
    }
}
```

---

## 9. Workshop: Product Catalog System

### 9.1 โครงสร้างระบบ

```
Product Catalog System
├── Categories (หมวดหมู่)
│   ├── id, name, slug, description, parent_id
├── Products (สินค้า)
│   ├── id, name, slug, description, price, compare_price
│   ├── stock_quantity, sku, image_url
│   ├── category_id, is_active, is_featured
├── Product Images (รูปภาพสินค้า)
│   ├── id, product_id, url, alt_text, sort_order
└── Product Reviews (รีวิว)
    ├── id, product_id, user_id, rating, comment
```

### 9.2 Migrations

```php
<?php
// database/migrations/create_categories_table.php

Schema::create('categories', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('slug')->unique();
    $table->text('description')->nullable();
    $table->unsignedBigInteger('parent_id')->nullable();
    $table->foreign('parent_id')->references('id')->on('categories')->onDelete('set null');
    $table->boolean('is_active')->default(true);
    $table->integer('sort_order')->default(0);
    $table->timestamps();
});

// database/migrations/create_product_images_table.php
Schema::create('product_images', function (Blueprint $table) {
    $table->id();
    $table->foreignId('product_id')->constrained()->onDelete('cascade');
    $table->string('url');
    $table->string('alt_text')->nullable();
    $table->integer('sort_order')->default(0);
    $table->boolean('is_primary')->default(false);
    $table->timestamps();
});

// database/migrations/create_product_reviews_table.php
Schema::create('product_reviews', function (Blueprint $table) {
    $table->id();
    $table->foreignId('product_id')->constrained()->onDelete('cascade');
    $table->foreignId('user_id')->constrained()->onDelete('cascade');
    $table->tinyInteger('rating'); // 1-5
    $table->string('title')->nullable();
    $table->text('comment')->nullable();
    $table->boolean('is_verified')->default(false);
    $table->timestamps();
    $table->unique(['product_id', 'user_id']); // 1 review per user per product
});
```

### 9.3 Models ครบระบบ

```php
<?php
// app/Models/Category.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Support\Str;

class Category extends Model
{
    use HasFactory;

    protected $fillable = ['name', 'slug', 'description', 'parent_id', 'is_active', 'sort_order'];

    protected $casts = [
        'is_active'  => 'boolean',
        'sort_order' => 'integer',
        'parent_id'  => 'integer',
    ];

    protected static function booted(): void
    {
        static::creating(function (Category $category) {
            if (empty($category->slug)) {
                $category->slug = Str::slug($category->name);
            }
        });
    }

    // Relationship
    public function products()
    {
        return $this->hasMany(Product::class);
    }

    public function parent()
    {
        return $this->belongsTo(Category::class, 'parent_id');
    }

    public function children()
    {
        return $this->hasMany(Category::class, 'parent_id');
    }

    // Scopes
    public function scopeActive($query)
    {
        return $query->where('is_active', true);
    }

    public function scopeParentOnly($query)
    {
        return $query->whereNull('parent_id');
    }
}
```

```php
<?php
// app/Models/Product.php (ฉบับเต็ม)

namespace App\Models;

use App\Enums\ProductStatus;
use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Support\Str;

class Product extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'name', 'slug', 'description', 'price', 'compare_price',
        'stock_quantity', 'sku', 'image_url', 'category_id',
        'is_active', 'is_featured',
    ];

    protected $casts = [
        'price'          => 'float',
        'compare_price'  => 'float',
        'stock_quantity' => 'integer',
        'is_active'      => 'boolean',
        'is_featured'    => 'boolean',
    ];

    protected $appends = ['formatted_price', 'discount_percentage', 'stock_status'];

    protected static function booted(): void
    {
        static::creating(function (Product $product) {
            if (empty($product->slug)) {
                $product->slug = Str::slug($product->name);
            }
            if (empty($product->sku)) {
                $product->sku = 'PROD-' . strtoupper(Str::random(8));
            }
        });

        static::updating(function (Product $product) {
            if ($product->isDirty('name') && empty($product->getOriginal('slug') !== $product->slug)) {
                $product->slug = Str::slug($product->name);
            }
        });
    }

    // Accessors
    protected function formattedPrice(): Attribute
    {
        return Attribute::make(
            get: fn ($value, $attributes) => '฿' . number_format($attributes['price'], 2),
        );
    }

    protected function discountPercentage(): Attribute
    {
        return Attribute::make(
            get: function ($value, $attributes) {
                if (empty($attributes['compare_price']) || $attributes['compare_price'] <= $attributes['price']) {
                    return 0;
                }
                return round((($attributes['compare_price'] - $attributes['price']) / $attributes['compare_price']) * 100);
            }
        );
    }

    protected function stockStatus(): Attribute
    {
        return Attribute::make(
            get: function ($value, $attributes) {
                $qty = $attributes['stock_quantity'];
                return match(true) {
                    $qty === 0  => 'out_of_stock',
                    $qty <= 5   => 'low_stock',
                    $qty <= 20  => 'limited',
                    default     => 'in_stock',
                };
            }
        );
    }

    // Relationships
    public function category()
    {
        return $this->belongsTo(Category::class);
    }

    public function images()
    {
        return $this->hasMany(ProductImage::class)->orderBy('sort_order');
    }

    public function primaryImage()
    {
        return $this->hasOne(ProductImage::class)->where('is_primary', true);
    }

    public function reviews()
    {
        return $this->hasMany(ProductReview::class);
    }

    // Scopes
    public function scopeActive($query)
    {
        return $query->where('is_active', true);
    }

    public function scopeFeatured($query)
    {
        return $query->where('is_featured', true);
    }

    public function scopeInStock($query)
    {
        return $query->where('stock_quantity', '>', 0);
    }

    public function scopeOfCategory($query, int $categoryId)
    {
        return $query->where('category_id', $categoryId);
    }

    public function scopePriceRange($query, float $min, float $max)
    {
        return $query->whereBetween('price', [$min, $max]);
    }

    public function scopeSearch($query, string $term)
    {
        return $query->where(function ($q) use ($term) {
            $q->where('name', 'like', "%{$term}%")
              ->orWhere('description', 'like', "%{$term}%")
              ->orWhere('sku', 'like', "%{$term}%");
        });
    }

    public function scopeOnSale($query)
    {
        return $query->whereNotNull('compare_price')
                     ->whereColumn('price', '<', 'compare_price');
    }
}
```

### 9.4 ProductController

```php
<?php
// app/Http/Controllers/ProductController.php

namespace App\Http\Controllers;

use App\Models\Product;
use App\Models\Category;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class ProductController extends Controller
{
    /**
     * แสดงรายการสินค้าพร้อม filter
     */
    public function index(Request $request): JsonResponse
    {
        $query = Product::active()->inStock();

        // Filter by category
        if ($request->has('category_id')) {
            $query->ofCategory($request->integer('category_id'));
        }

        // Filter by price range
        if ($request->has('min_price') && $request->has('max_price')) {
            $query->priceRange(
                $request->float('min_price'),
                $request->float('max_price')
            );
        }

        // Search
        if ($request->has('search')) {
            $query->search($request->string('search'));
        }

        // Filter featured
        if ($request->boolean('featured')) {
            $query->featured();
        }

        // Filter on sale
        if ($request->boolean('on_sale')) {
            $query->onSale();
        }

        // Sort
        $sortField = $request->get('sort', 'created_at');
        $sortDir = $request->get('order', 'desc');
        $query->orderBy($sortField, $sortDir);

        // Paginate
        $perPage = min($request->integer('per_page', 15), 100);
        $products = $query->with(['category', 'primaryImage'])->paginate($perPage);

        return response()->json($products);
    }

    /**
     * สร้างสินค้าใหม่
     */
    public function store(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'name'           => 'required|string|max:255',
            'description'    => 'nullable|string',
            'price'          => 'required|numeric|min:0',
            'compare_price'  => 'nullable|numeric|min:0|gt:price',
            'stock_quantity' => 'required|integer|min:0',
            'category_id'    => 'required|exists:categories,id',
            'is_active'      => 'boolean',
            'is_featured'    => 'boolean',
        ]);

        $product = Product::create($validated);

        return response()->json([
            'message' => 'สร้างสินค้าสำเร็จ',
            'product' => $product->load(['category']),
        ], 201);
    }

    /**
     * แสดงรายละเอียดสินค้า
     */
    public function show(int $id): JsonResponse
    {
        $product = Product::active()
            ->with(['category', 'images', 'reviews.user'])
            ->findOrFail($id);

        return response()->json($product);
    }

    /**
     * อัพเดตสินค้า
     */
    public function update(Request $request, int $id): JsonResponse
    {
        $product = Product::findOrFail($id);

        $validated = $request->validate([
            'name'           => 'sometimes|string|max:255',
            'description'    => 'nullable|string',
            'price'          => 'sometimes|numeric|min:0',
            'compare_price'  => 'nullable|numeric|min:0',
            'stock_quantity' => 'sometimes|integer|min:0',
            'category_id'    => 'sometimes|exists:categories,id',
            'is_active'      => 'boolean',
            'is_featured'    => 'boolean',
        ]);

        $product->update($validated);

        return response()->json([
            'message' => 'อัพเดตสินค้าสำเร็จ',
            'product' => $product->fresh()->load(['category']),
        ]);
    }

    /**
     * ลบสินค้า (soft delete)
     */
    public function destroy(int $id): JsonResponse
    {
        $product = Product::findOrFail($id);
        $product->delete();

        return response()->json([
            'message' => 'ลบสินค้าสำเร็จ',
        ]);
    }

    /**
     * อัพเดต stock
     */
    public function updateStock(Request $request, int $id): JsonResponse
    {
        $product = Product::findOrFail($id);

        $request->validate([
            'action'   => 'required|in:add,subtract,set',
            'quantity' => 'required|integer|min:0',
        ]);

        match($request->action) {
            'add'      => $product->increment('stock_quantity', $request->quantity),
            'subtract' => $product->decrement('stock_quantity', $request->quantity),
            'set'      => $product->update(['stock_quantity' => $request->quantity]),
        };

        return response()->json([
            'message'        => 'อัพเดต stock สำเร็จ',
            'stock_quantity' => $product->fresh()->stock_quantity,
            'stock_status'   => $product->fresh()->stock_status,
        ]);
    }
}
```

### 9.5 Factory & Seeder

```php
<?php
// database/factories/ProductFactory.php

namespace Database\Factories;

use App\Models\Category;
use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Str;

class ProductFactory extends Factory
{
    public function definition(): array
    {
        $name = $this->faker->unique()->words(3, true);
        $price = $this->faker->randomFloat(2, 100, 50000);
        
        return [
            'name'           => ucwords($name),
            'slug'           => Str::slug($name),
            'description'    => $this->faker->paragraphs(2, true),
            'price'          => $price,
            'compare_price'  => $this->faker->boolean(50) ? $price * 1.2 : null,
            'stock_quantity' => $this->faker->numberBetween(0, 100),
            'sku'            => 'PROD-' . strtoupper(Str::random(8)),
            'category_id'    => Category::inRandomOrder()->value('id'),
            'is_active'      => $this->faker->boolean(80),
            'is_featured'    => $this->faker->boolean(20),
        ];
    }

    public function active(): static
    {
        return $this->state(['is_active' => true]);
    }

    public function featured(): static
    {
        return $this->state(['is_featured' => true, 'is_active' => true]);
    }

    public function outOfStock(): static
    {
        return $this->state(['stock_quantity' => 0]);
    }

    public function onSale(): static
    {
        return $this->state(function (array $attributes) {
            return [
                'compare_price' => $attributes['price'] * 1.3,
            ];
        });
    }
}
```

```php
<?php
// database/seeders/ProductSeeder.php

namespace Database\Seeders;

use App\Models\Category;
use App\Models\Product;
use Illuminate\Database\Seeder;

class ProductSeeder extends Seeder
{
    public function run(): void
    {
        // สร้าง categories
        $categories = Category::factory(5)->create();

        // สร้าง products
        Product::factory(50)->active()->create();
        Product::factory(10)->featured()->create();
        Product::factory(5)->outOfStock()->create();
        Product::factory(15)->onSale()->create();

        $this->command->info('✅ สร้าง products สำเร็จ');
    }
}
```

---

## Quiz พร้อมเฉลย

### คำถามที่ 1
จงเขียน query ที่ดึงสินค้าที่ active, มี stock, ราคาระหว่าง 500-5000 บาท เรียงตามราคาจากน้อยไปมาก และ paginate 10 items

**เฉลย:**
```php
$products = Product::active()
    ->inStock()
    ->priceRange(500, 5000)
    ->orderBy('price', 'asc')
    ->paginate(10);
```

### คำถามที่ 2
อธิบายความแตกต่างระหว่าง `$fillable` และ `$guarded`

**เฉลย:**
- `$fillable` คือ whitelist - ระบุ fields ที่ **อนุญาต** ให้ทำ mass assignment
- `$guarded` คือ blacklist - ระบุ fields ที่ **ไม่อนุญาต** ให้ทำ mass assignment
- ใช้อย่างใดอย่างหนึ่ง ไม่ควรใช้ทั้งสอง
- `$guarded = []` หมายความว่า อนุญาตทุก field (ไม่ปลอดภัย!)

### คำถามที่ 3
อธิบาย Soft Delete และเขียนโค้ดตัวอย่าง

**เฉลย:**
Soft Delete คือการลบข้อมูลแบบ "ซ่อน" ไม่ได้ลบจริงจากฐานข้อมูล แต่จะ set `deleted_at` เป็น timestamp

```php
// ต้องเพิ่ม SoftDeletes trait และ column ใน migration
Schema::table('products', function (Blueprint $table) {
    $table->softDeletes(); // เพิ่ม deleted_at column
});

// ใน Model
use Illuminate\Database\Eloquent\SoftDeletes;
class Product extends Model {
    use SoftDeletes;
}

// Soft delete
$product->delete(); // deleted_at = now()

// ดึงที่ถูก delete ด้วย
Product::withTrashed()->get();

// Restore
$product->restore(); // deleted_at = null

// Force delete
$product->forceDelete();
```

### คำถามที่ 4
อะไรคือความแตกต่างระหว่าง `first()` และ `firstOrFail()`?

**เฉลย:**
- `first()` คืน `null` ถ้าไม่พบข้อมูล
- `firstOrFail()` throw `Illuminate\Database\Eloquent\ModelNotFoundException` ถ้าไม่พบข้อมูล ซึ่ง Laravel จะแปลงเป็น 404 HTTP response อัตโนมัติ

### คำถามที่ 5
Accessor และ Mutator ต่างกันอย่างไร?

**เฉลย:**
- **Accessor** ทำงานเมื่อ **อ่าน** ข้อมูล (GET) - แปลงค่าที่ดึงมาจากฐานข้อมูล
- **Mutator** ทำงานเมื่อ **เขียน** ข้อมูล (SET) - แปลงค่าก่อนบันทึกลงฐานข้อมูล
- ใน Laravel 9+ ใช้ `Attribute::make(get: ..., set: ...)` ใน method เดียว

---

## แบบฝึกหัด

### Exercise 1: สร้าง Category Model
สร้าง Category Model ที่มี:
- Self-referencing relationship (parent/children)
- Scope สำหรับ active categories
- Scope สำหรับ root categories เท่านั้น
- Accessor สำหรับ full path (เช่น "Electronics > Smartphones")

### Exercise 2: สร้าง ProductReview Model
สร้าง ProductReview Model ที่มี:
- Relationship กับ Product และ User
- Scope สำหรับ verified reviews
- Accessor สำหรับ formatted rating (5 ดาว)

### Exercise 3: Catalog Statistics
เขียน query เพื่อดึง:
1. จำนวนสินค้าแต่ละ category
2. สินค้าที่ขายดีที่สุด (มีใน order มากที่สุด)
3. มูลค่า stock รวมทั้งหมด

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Model Creation | make:model, table naming, timestamps |
| CRUD | create, read, update, delete methods |
| Query Builder | where, join, aggregate, pagination |
| Scopes | Local scope, Global scope |
| Mutators/Accessors | Attribute::make(), get/set |
| Casts | Built-in, Custom, Enum |

---

## ลิงก์ไป Part ถัดไป

➡️ [Part 032: Laravel Eloquent Relationships](part-032-laravel-eloquent-relationships.md)

ใน Part ถัดไปเราจะเรียนเรื่อง Relationships ระหว่าง Models ซึ่งเป็นหนึ่งในความสามารถที่ทรงพลังที่สุดของ Eloquent
