# Part 046: Laravel Package Development — สร้าง Package ของตัวเอง

**ระดับ: ระดับโลก (World-class)**
**เวลาเรียน: 6-8 ชั่วโมง**

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เข้าใจโครงสร้าง Laravel Package
- สร้าง Service Provider และ Facade
- จัดการ Config, Views, Migrations
- เพิ่ม Artisan Commands ใน Package
- เขียน Tests สำหรับ Package
- สร้าง Laravel Package สำหรับ Thai Address

---

## 1. ทำไมต้องสร้าง Package?

**เหตุผลในการสร้าง Package:**
- แบ่งปัน Code ระหว่างหลาย Projects
- เผยแพร่ Open Source บน Packagist
- จัดการ Code แบบ Modular
- ขาย Premium Package บน marketplace

**Package Types:**
- **Library:** ฟังก์ชันพื้นฐาน ไม่พึ่งพา Framework
- **Laravel Package:** ออกแบบเฉพาะสำหรับ Laravel
- **Micro-package:** Package ขนาดเล็กทำสิ่งเดียว

---

## 2. โครงสร้าง Package

```
thai-address/
├── src/
│   ├── ThaiAddressServiceProvider.php
│   ├── ThaiAddress.php
│   ├── Facades/
│   │   └── ThaiAddress.php
│   ├── Models/
│   │   ├── Province.php
│   │   ├── District.php
│   │   └── SubDistrict.php
│   ├── Http/
│   │   └── Controllers/
│   │       └── AddressController.php
│   ├── Commands/
│   │   └── ImportAddressData.php
│   └── Traits/
│       └── HasThaiAddress.php
├── database/
│   ├── migrations/
│   │   └── create_thai_addresses_tables.php
│   └── seeders/
│       └── ThaiAddressSeeder.php
├── config/
│   └── thai-address.php
├── resources/
│   ├── views/
│   │   └── address-selector.blade.php
│   ├── js/
│   │   └── address-selector.js
│   └── data/
│       └── thailand.json
├── routes/
│   └── api.php
├── tests/
│   ├── Feature/
│   └── Unit/
├── composer.json
├── README.md
└── CHANGELOG.md
```

---

## 3. สร้าง Package Structure

### 3.1 เริ่มต้นด้วย composer.json

```json
{
    "name": "yourname/thai-address",
    "description": "Thai Address (Province, District, Sub-district) for Laravel",
    "type": "library",
    "keywords": ["laravel", "thai", "address", "province", "district"],
    "license": "MIT",
    "authors": [
        {
            "name": "Your Name",
            "email": "your@email.com"
        }
    ],
    "require": {
        "php": "^8.1",
        "illuminate/support": "^10.0|^11.0",
        "illuminate/database": "^10.0|^11.0"
    },
    "require-dev": {
        "orchestra/testbench": "^8.0|^9.0",
        "phpunit/phpunit": "^10.0"
    },
    "autoload": {
        "psr-4": {
            "YourName\\ThaiAddress\\": "src/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "YourName\\ThaiAddress\\Tests\\": "tests/"
        }
    },
    "extra": {
        "laravel": {
            "providers": [
                "YourName\\ThaiAddress\\ThaiAddressServiceProvider"
            ],
            "aliases": {
                "ThaiAddress": "YourName\\ThaiAddress\\Facades\\ThaiAddress"
            }
        }
    },
    "minimum-stability": "dev",
    "prefer-stable": true
}
```

---

## 4. Service Provider

Service Provider เป็นหัวใจของ Laravel Package

```php
// src/ThaiAddressServiceProvider.php
namespace YourName\ThaiAddress;

use Illuminate\Support\ServiceProvider;

class ThaiAddressServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Merge Package Config
        $this->mergeConfigFrom(
            __DIR__ . '/../config/thai-address.php',
            'thai-address'
        );
        
        // Register Main Service
        $this->app->singleton('thai-address', function ($app) {
            return new ThaiAddress(
                config('thai-address'),
                $app->make(\Illuminate\Database\DatabaseManager::class)
            );
        });
        
        // Register Alias
        $this->app->alias('thai-address', ThaiAddress::class);
    }
    
    public function boot(): void
    {
        // Publish Config
        if ($this->app->runningInConsole()) {
            $this->publishes([
                __DIR__ . '/../config/thai-address.php' => config_path('thai-address.php'),
            ], 'thai-address-config');
            
            // Publish Migrations
            $this->publishes([
                __DIR__ . '/../database/migrations/' => database_path('migrations'),
            ], 'thai-address-migrations');
            
            // Publish Seeders
            $this->publishes([
                __DIR__ . '/../database/seeders/' => database_path('seeders'),
            ], 'thai-address-seeders');
            
            // Publish Views
            $this->publishes([
                __DIR__ . '/../resources/views/' => resource_path('views/vendor/thai-address'),
            ], 'thai-address-views');
            
            // Publish Assets
            $this->publishes([
                __DIR__ . '/../resources/js/' => public_path('vendor/thai-address/js'),
            ], 'thai-address-assets');
            
            // Register Commands
            $this->commands([
                \YourName\ThaiAddress\Commands\ImportAddressData::class,
            ]);
        }
        
        // Load Views
        $this->loadViewsFrom(__DIR__ . '/../resources/views', 'thai-address');
        
        // Load Translations
        $this->loadTranslationsFrom(__DIR__ . '/../lang', 'thai-address');
        
        // Load Routes
        $this->loadRoutesFrom(__DIR__ . '/../routes/api.php');
        
        // Load Migrations (auto-run)
        $this->loadMigrationsFrom(__DIR__ . '/../database/migrations');
    }
}
```

---

## 5. Facades

Facade ให้ Static-like interface กับ Service

```php
// src/Facades/ThaiAddress.php
namespace YourName\ThaiAddress\Facades;

use Illuminate\Support\Facades\Facade;

/**
 * @method static \Illuminate\Support\Collection getProvinces()
 * @method static \Illuminate\Support\Collection getDistricts(int $provinceId)
 * @method static \Illuminate\Support\Collection getSubDistricts(int $districtId)
 * @method static array|null findByZipCode(string $zipCode)
 * @method static string formatAddress(array $addressData)
 *
 * @see \YourName\ThaiAddress\ThaiAddress
 */
class ThaiAddress extends Facade
{
    protected static function getFacadeAccessor(): string
    {
        return 'thai-address';
    }
}
```

```php
// src/ThaiAddress.php
namespace YourName\ThaiAddress;

use Illuminate\Support\Collection;
use Illuminate\Database\DatabaseManager;

class ThaiAddress
{
    public function __construct(
        private readonly array $config,
        private readonly DatabaseManager $db
    ) {}
    
    public function getProvinces(): Collection
    {
        return $this->db->table($this->config['tables']['provinces'])
            ->orderBy('name_th')
            ->get();
    }
    
    public function getDistricts(int $provinceId): Collection
    {
        return $this->db->table($this->config['tables']['districts'])
            ->where('province_id', $provinceId)
            ->orderBy('name_th')
            ->get();
    }
    
    public function getSubDistricts(int $districtId): Collection
    {
        return $this->db->table($this->config['tables']['sub_districts'])
            ->where('district_id', $districtId)
            ->orderBy('name_th')
            ->get();
    }
    
    public function findByZipCode(string $zipCode): ?array
    {
        $subDistrict = $this->db->table($this->config['tables']['sub_districts'])
            ->where('zip_code', $zipCode)
            ->first();
        
        if (!$subDistrict) {
            return null;
        }
        
        $district = $this->db->table($this->config['tables']['districts'])
            ->find($subDistrict->district_id);
        
        $province = $this->db->table($this->config['tables']['provinces'])
            ->find($district->province_id);
        
        return [
            'sub_district' => $subDistrict,
            'district' => $district,
            'province' => $province,
            'zip_code' => $zipCode,
        ];
    }
    
    public function search(string $query): Collection
    {
        return $this->db->table($this->config['tables']['sub_districts'] . ' as sd')
            ->join($this->config['tables']['districts'] . ' as d', 'd.id', '=', 'sd.district_id')
            ->join($this->config['tables']['provinces'] . ' as p', 'p.id', '=', 'd.province_id')
            ->where('sd.name_th', 'like', "%{$query}%")
            ->orWhere('sd.zip_code', 'like', "%{$query}%")
            ->select('sd.*', 'd.name_th as district_name', 'p.name_th as province_name')
            ->get();
    }
    
    public function formatAddress(array $data): string
    {
        $parts = array_filter([
            $data['address'] ?? null,
            $data['sub_district'] ?? null,
            $data['district'] ? ('อำเภอ' . $data['district']) : null,
            $data['province'] ? ('จังหวัด' . $data['province']) : null,
            $data['zip_code'] ?? null,
        ]);
        
        return implode(' ', $parts);
    }
}
```

---

## 6. Config File

```php
// config/thai-address.php
return [
    /*
    |--------------------------------------------------------------------------
    | Table Names
    |--------------------------------------------------------------------------
    */
    'tables' => [
        'provinces' => 'th_provinces',
        'districts' => 'th_districts',
        'sub_districts' => 'th_sub_districts',
    ],
    
    /*
    |--------------------------------------------------------------------------
    | Cache Settings
    |--------------------------------------------------------------------------
    */
    'cache' => [
        'enabled' => env('THAI_ADDRESS_CACHE', true),
        'ttl' => env('THAI_ADDRESS_CACHE_TTL', 86400), // 24 hours
        'prefix' => 'thai_address',
    ],
    
    /*
    |--------------------------------------------------------------------------
    | Language
    |--------------------------------------------------------------------------
    */
    'default_language' => env('THAI_ADDRESS_LANG', 'th'), // th, en
    
    /*
    |--------------------------------------------------------------------------
    | API Settings
    |--------------------------------------------------------------------------
    */
    'api' => [
        'enabled' => true,
        'prefix' => 'api/thai-address',
        'middleware' => ['api'],
    ],
];
```

---

## 7. Models

```php
// src/Models/Province.php
namespace YourName\ThaiAddress\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Province extends Model
{
    protected $fillable = ['name_th', 'name_en', 'code'];
    
    public function getTable(): string
    {
        return config('thai-address.tables.provinces');
    }
    
    public function districts(): HasMany
    {
        return $this->hasMany(District::class);
    }
    
    public function scopeByRegion($query, string $region)
    {
        return $query->where('region', $region);
    }
}
```

```php
// src/Models/District.php
namespace YourName\ThaiAddress\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class District extends Model
{
    protected $fillable = ['province_id', 'name_th', 'name_en', 'code'];
    
    public function getTable(): string
    {
        return config('thai-address.tables.districts');
    }
    
    public function province(): BelongsTo
    {
        return $this->belongsTo(Province::class);
    }
    
    public function subDistricts(): HasMany
    {
        return $this->hasMany(SubDistrict::class);
    }
}
```

```php
// src/Models/SubDistrict.php
namespace YourName\ThaiAddress\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class SubDistrict extends Model
{
    protected $fillable = ['district_id', 'name_th', 'name_en', 'zip_code', 'lat', 'lng'];
    
    public function getTable(): string
    {
        return config('thai-address.tables.sub_districts');
    }
    
    public function district(): BelongsTo
    {
        return $this->belongsTo(District::class);
    }
}
```

---

## 8. Migrations

```php
// database/migrations/2024_01_01_000001_create_thai_addresses_tables.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('th_provinces', function (Blueprint $table) {
            $table->id();
            $table->string('name_th');
            $table->string('name_en');
            $table->string('code', 10)->unique();
            $table->string('region', 20)->nullable(); // north, northeast, east, central, west, south
            $table->timestamps();
            
            $table->index('code');
            $table->index('region');
        });
        
        Schema::create('th_districts', function (Blueprint $table) {
            $table->id();
            $table->foreignId('province_id')->constrained('th_provinces')->onDelete('cascade');
            $table->string('name_th');
            $table->string('name_en');
            $table->string('code', 10)->nullable();
            $table->timestamps();
            
            $table->index('province_id');
            $table->index('code');
        });
        
        Schema::create('th_sub_districts', function (Blueprint $table) {
            $table->id();
            $table->foreignId('district_id')->constrained('th_districts')->onDelete('cascade');
            $table->string('name_th');
            $table->string('name_en');
            $table->string('zip_code', 10)->nullable();
            $table->decimal('lat', 10, 8)->nullable();
            $table->decimal('lng', 11, 8)->nullable();
            $table->timestamps();
            
            $table->index('district_id');
            $table->index('zip_code');
        });
    }
    
    public function down(): void
    {
        Schema::dropIfExists('th_sub_districts');
        Schema::dropIfExists('th_districts');
        Schema::dropIfExists('th_provinces');
    }
};
```

---

## 9. Artisan Commands

```php
// src/Commands/ImportAddressData.php
namespace YourName\ThaiAddress\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;

class ImportAddressData extends Command
{
    protected $signature = 'thai-address:import 
                            {--force : Overwrite existing data}
                            {--file= : Path to custom data file}';
    
    protected $description = 'Import Thai address data (Provinces, Districts, Sub-districts)';
    
    public function handle(): int
    {
        if (!$this->option('force') && $this->dataExists()) {
            if (!$this->confirm('Data already exists. Do you want to overwrite it?')) {
                $this->info('Import cancelled.');
                return 0;
            }
        }
        
        $dataFile = $this->option('file') ?: __DIR__ . '/../../resources/data/thailand.json';
        
        if (!file_exists($dataFile)) {
            $this->error("Data file not found: {$dataFile}");
            return 1;
        }
        
        $this->info('Importing Thai address data...');
        
        $data = json_decode(file_get_contents($dataFile), true);
        
        if (json_last_error() !== JSON_ERROR_NONE) {
            $this->error('Invalid JSON data file.');
            return 1;
        }
        
        DB::transaction(function () use ($data) {
            $this->truncateTables();
            $this->importProvinces($data['provinces'] ?? []);
        });
        
        $this->info('Thai address data imported successfully!');
        $this->table(
            ['Table', 'Records'],
            [
                ['Provinces', DB::table('th_provinces')->count()],
                ['Districts', DB::table('th_districts')->count()],
                ['Sub-districts', DB::table('th_sub_districts')->count()],
            ]
        );
        
        return 0;
    }
    
    private function dataExists(): bool
    {
        return DB::table('th_provinces')->exists();
    }
    
    private function truncateTables(): void
    {
        DB::statement('SET FOREIGN_KEY_CHECKS=0');
        DB::table('th_sub_districts')->truncate();
        DB::table('th_districts')->truncate();
        DB::table('th_provinces')->truncate();
        DB::statement('SET FOREIGN_KEY_CHECKS=1');
    }
    
    private function importProvinces(array $provinces): void
    {
        $bar = $this->output->createProgressBar(count($provinces));
        $bar->start();
        
        foreach ($provinces as $provinceData) {
            $province = DB::table('th_provinces')->insertGetId([
                'name_th' => $provinceData['name_th'],
                'name_en' => $provinceData['name_en'],
                'code' => $provinceData['code'],
                'region' => $provinceData['region'] ?? null,
                'created_at' => now(),
                'updated_at' => now(),
            ]);
            
            foreach ($provinceData['districts'] ?? [] as $districtData) {
                $district = DB::table('th_districts')->insertGetId([
                    'province_id' => $province,
                    'name_th' => $districtData['name_th'],
                    'name_en' => $districtData['name_en'],
                    'code' => $districtData['code'] ?? null,
                    'created_at' => now(),
                    'updated_at' => now(),
                ]);
                
                $subDistricts = array_map(fn($sub) => [
                    'district_id' => $district,
                    'name_th' => $sub['name_th'],
                    'name_en' => $sub['name_en'],
                    'zip_code' => $sub['zip_code'] ?? null,
                    'lat' => $sub['lat'] ?? null,
                    'lng' => $sub['lng'] ?? null,
                    'created_at' => now(),
                    'updated_at' => now(),
                ], $districtData['sub_districts'] ?? []);
                
                // Insert แบบ Batch
                foreach (array_chunk($subDistricts, 100) as $chunk) {
                    DB::table('th_sub_districts')->insert($chunk);
                }
            }
            
            $bar->advance();
        }
        
        $bar->finish();
        $this->newLine();
    }
}
```

---

## 10. Traits

```php
// src/Traits/HasThaiAddress.php
namespace YourName\ThaiAddress\Traits;

use YourName\ThaiAddress\Models\Province;
use YourName\ThaiAddress\Models\District;
use YourName\ThaiAddress\Models\SubDistrict;

trait HasThaiAddress
{
    public function province()
    {
        return $this->belongsTo(Province::class, 'province_id');
    }
    
    public function district()
    {
        return $this->belongsTo(District::class, 'district_id');
    }
    
    public function subDistrict()
    {
        return $this->belongsTo(SubDistrict::class, 'sub_district_id');
    }
    
    public function getFullAddressAttribute(): string
    {
        $parts = array_filter([
            $this->address ?? null,
            $this->subDistrict?->name_th ? 'ตำบล' . $this->subDistrict->name_th : null,
            $this->district?->name_th ? 'อำเภอ' . $this->district->name_th : null,
            $this->province?->name_th ? 'จังหวัด' . $this->province->name_th : null,
            $this->zip_code ?? null,
        ]);
        
        return implode(' ', $parts);
    }
    
    public function scopeInProvince($query, int $provinceId)
    {
        return $query->where('province_id', $provinceId);
    }
    
    public function scopeInDistrict($query, int $districtId)
    {
        return $query->where('district_id', $districtId);
    }
}
```

---

## 11. Views และ JavaScript Component

```blade
{{-- resources/views/address-selector.blade.php --}}
<div
    class="thai-address-selector"
    x-data="thaiAddressSelector({
        provinceId: @js($provinceId ?? null),
        districtId: @js($districtId ?? null),
        subDistrictId: @js($subDistrictId ?? null),
        apiPrefix: '{{ config('thai-address.api.prefix') }}'
    })"
>
    {{-- Province --}}
    <div class="field-group">
        <label>จังหวัด</label>
        <select
            name="{{ $names['province'] ?? 'province_id' }}"
            x-model="selectedProvince"
            @change="loadDistricts()"
        >
            <option value="">-- เลือกจังหวัด --</option>
            <template x-for="province in provinces" :key="province.id">
                <option :value="province.id" x-text="province.name_th"></option>
            </template>
        </select>
    </div>
    
    {{-- District --}}
    <div class="field-group">
        <label>อำเภอ/เขต</label>
        <select
            name="{{ $names['district'] ?? 'district_id' }}"
            x-model="selectedDistrict"
            @change="loadSubDistricts()"
            :disabled="!selectedProvince"
        >
            <option value="">-- เลือกอำเภอ --</option>
            <template x-for="district in districts" :key="district.id">
                <option :value="district.id" x-text="district.name_th"></option>
            </template>
        </select>
    </div>
    
    {{-- Sub-district --}}
    <div class="field-group">
        <label>ตำบล/แขวง</label>
        <select
            name="{{ $names['sub_district'] ?? 'sub_district_id' }}"
            x-model="selectedSubDistrict"
            @change="updateZipCode()"
            :disabled="!selectedDistrict"
        >
            <option value="">-- เลือกตำบล --</option>
            <template x-for="subDistrict in subDistricts" :key="subDistrict.id">
                <option :value="subDistrict.id" x-text="subDistrict.name_th"></option>
            </template>
        </select>
    </div>
    
    {{-- Zip Code --}}
    <div class="field-group">
        <label>รหัสไปรษณีย์</label>
        <input
            type="text"
            name="{{ $names['zip_code'] ?? 'zip_code' }}"
            x-model="zipCode"
            readonly
            placeholder="Auto-filled"
        >
    </div>
</div>
```

```javascript
// resources/js/address-selector.js
function thaiAddressSelector({ provinceId, districtId, subDistrictId, apiPrefix }) {
    return {
        provinces: [],
        districts: [],
        subDistricts: [],
        selectedProvince: provinceId,
        selectedDistrict: districtId,
        selectedSubDistrict: subDistrictId,
        zipCode: '',
        
        async init() {
            await this.loadProvinces();
            
            if (this.selectedProvince) {
                await this.loadDistricts();
            }
            
            if (this.selectedDistrict) {
                await this.loadSubDistricts();
            }
            
            if (this.selectedSubDistrict) {
                this.updateZipCode();
            }
        },
        
        async loadProvinces() {
            const response = await fetch(`/${apiPrefix}/provinces`);
            this.provinces = await response.json();
        },
        
        async loadDistricts() {
            if (!this.selectedProvince) {
                this.districts = [];
                this.selectedDistrict = null;
                this.subDistricts = [];
                this.selectedSubDistrict = null;
                return;
            }
            
            const response = await fetch(`/${apiPrefix}/provinces/${this.selectedProvince}/districts`);
            this.districts = await response.json();
            this.selectedDistrict = null;
            this.subDistricts = [];
            this.selectedSubDistrict = null;
        },
        
        async loadSubDistricts() {
            if (!this.selectedDistrict) {
                this.subDistricts = [];
                this.selectedSubDistrict = null;
                return;
            }
            
            const response = await fetch(`/${apiPrefix}/districts/${this.selectedDistrict}/sub-districts`);
            this.subDistricts = await response.json();
            this.selectedSubDistrict = null;
        },
        
        updateZipCode() {
            const subDistrict = this.subDistricts.find(s => s.id == this.selectedSubDistrict);
            this.zipCode = subDistrict?.zip_code ?? '';
        },
    };
}
```

---

## 12. API Routes

```php
// routes/api.php
use YourName\ThaiAddress\Http\Controllers\AddressController;

Route::group([
    'prefix' => config('thai-address.api.prefix', 'api/thai-address'),
    'middleware' => config('thai-address.api.middleware', ['api']),
], function () {
    Route::get('/provinces', [AddressController::class, 'provinces']);
    Route::get('/provinces/{province}/districts', [AddressController::class, 'districts']);
    Route::get('/districts/{district}/sub-districts', [AddressController::class, 'subDistricts']);
    Route::get('/sub-districts/search', [AddressController::class, 'search']);
    Route::get('/zip-code/{zipCode}', [AddressController::class, 'findByZipCode']);
});
```

```php
// src/Http/Controllers/AddressController.php
namespace YourName\ThaiAddress\Http\Controllers;

use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Routing\Controller;
use YourName\ThaiAddress\Facades\ThaiAddress;

class AddressController extends Controller
{
    public function provinces(): JsonResponse
    {
        $provinces = cache()->remember(
            'thai-address:provinces',
            config('thai-address.cache.ttl'),
            fn() => ThaiAddress::getProvinces()
        );
        
        return response()->json($provinces);
    }
    
    public function districts(int $provinceId): JsonResponse
    {
        $districts = cache()->remember(
            "thai-address:districts:{$provinceId}",
            config('thai-address.cache.ttl'),
            fn() => ThaiAddress::getDistricts($provinceId)
        );
        
        return response()->json($districts);
    }
    
    public function subDistricts(int $districtId): JsonResponse
    {
        $subDistricts = cache()->remember(
            "thai-address:sub-districts:{$districtId}",
            config('thai-address.cache.ttl'),
            fn() => ThaiAddress::getSubDistricts($districtId)
        );
        
        return response()->json($subDistricts);
    }
    
    public function search(Request $request): JsonResponse
    {
        $query = $request->get('q', '');
        
        if (strlen($query) < 2) {
            return response()->json([]);
        }
        
        return response()->json(ThaiAddress::search($query));
    }
    
    public function findByZipCode(string $zipCode): JsonResponse
    {
        $result = ThaiAddress::findByZipCode($zipCode);
        
        if (!$result) {
            return response()->json(['error' => 'Zip code not found'], 404);
        }
        
        return response()->json($result);
    }
}
```

---

## 13. Testing Package

### 13.1 Setup TestCase

```php
// tests/TestCase.php
namespace YourName\ThaiAddress\Tests;

use Orchestra\Testbench\TestCase as OrchestraTestCase;
use YourName\ThaiAddress\ThaiAddressServiceProvider;

abstract class TestCase extends OrchestraTestCase
{
    protected function getPackageProviders($app): array
    {
        return [
            ThaiAddressServiceProvider::class,
        ];
    }
    
    protected function getPackageAliases($app): array
    {
        return [
            'ThaiAddress' => \YourName\ThaiAddress\Facades\ThaiAddress::class,
        ];
    }
    
    protected function getEnvironmentSetUp($app): void
    {
        // Database setup
        $app['config']->set('database.default', 'testing');
        $app['config']->set('database.connections.testing', [
            'driver' => 'sqlite',
            'database' => ':memory:',
            'prefix' => '',
        ]);
    }
    
    protected function setUp(): void
    {
        parent::setUp();
        
        // Run migrations
        $this->loadMigrationsFrom(__DIR__ . '/../database/migrations');
        
        // Seed test data
        $this->seedTestData();
    }
    
    private function seedTestData(): void
    {
        \Illuminate\Support\Facades\DB::table('th_provinces')->insert([
            ['name_th' => 'กรุงเทพมหานคร', 'name_en' => 'Bangkok', 'code' => '10', 'created_at' => now(), 'updated_at' => now()],
            ['name_th' => 'เชียงใหม่', 'name_en' => 'Chiang Mai', 'code' => '50', 'created_at' => now(), 'updated_at' => now()],
        ]);
        
        \Illuminate\Support\Facades\DB::table('th_districts')->insert([
            ['province_id' => 1, 'name_th' => 'พระนคร', 'name_en' => 'Phra Nakhon', 'created_at' => now(), 'updated_at' => now()],
            ['province_id' => 2, 'name_th' => 'เมืองเชียงใหม่', 'name_en' => 'Mueang Chiang Mai', 'created_at' => now(), 'updated_at' => now()],
        ]);
        
        \Illuminate\Support\Facades\DB::table('th_sub_districts')->insert([
            ['district_id' => 1, 'name_th' => 'พระบรมมหาราชวัง', 'name_en' => 'Phra Borom Maha Ratchawang', 'zip_code' => '10200', 'created_at' => now(), 'updated_at' => now()],
        ]);
    }
}
```

### 13.2 Unit Tests

```php
// tests/Unit/ThaiAddressTest.php
namespace YourName\ThaiAddress\Tests\Unit;

use YourName\ThaiAddress\Tests\TestCase;
use YourName\ThaiAddress\Facades\ThaiAddress;

class ThaiAddressTest extends TestCase
{
    public function test_can_get_all_provinces(): void
    {
        $provinces = ThaiAddress::getProvinces();
        
        $this->assertCount(2, $provinces);
        $this->assertEquals('กรุงเทพมหานคร', $provinces->first()->name_th);
    }
    
    public function test_can_get_districts_by_province(): void
    {
        $districts = ThaiAddress::getDistricts(1);
        
        $this->assertCount(1, $districts);
        $this->assertEquals('พระนคร', $districts->first()->name_th);
    }
    
    public function test_can_find_by_zip_code(): void
    {
        $result = ThaiAddress::findByZipCode('10200');
        
        $this->assertNotNull($result);
        $this->assertEquals('10200', $result['zip_code']);
        $this->assertEquals('พระบรมมหาราชวัง', $result['sub_district']->name_th);
    }
    
    public function test_returns_null_for_invalid_zip_code(): void
    {
        $result = ThaiAddress::findByZipCode('99999');
        
        $this->assertNull($result);
    }
    
    public function test_format_address(): void
    {
        $address = ThaiAddress::formatAddress([
            'address' => '123 ถนนราชดำเนิน',
            'sub_district' => 'พระบรมมหาราชวัง',
            'district' => 'พระนคร',
            'province' => 'กรุงเทพมหานคร',
            'zip_code' => '10200',
        ]);
        
        $this->assertStringContainsString('123 ถนนราชดำเนิน', $address);
        $this->assertStringContainsString('กรุงเทพมหานคร', $address);
        $this->assertStringContainsString('10200', $address);
    }
}
```

### 13.3 Feature Tests

```php
// tests/Feature/AddressApiTest.php
namespace YourName\ThaiAddress\Tests\Feature;

use YourName\ThaiAddress\Tests\TestCase;

class AddressApiTest extends TestCase
{
    public function test_can_get_provinces_via_api(): void
    {
        $response = $this->getJson('/api/thai-address/provinces');
        
        $response->assertStatus(200);
        $response->assertJsonCount(2);
        $response->assertJsonFragment(['name_th' => 'กรุงเทพมหานคร']);
    }
    
    public function test_can_get_districts_via_api(): void
    {
        $response = $this->getJson('/api/thai-address/provinces/1/districts');
        
        $response->assertStatus(200);
        $response->assertJsonCount(1);
    }
    
    public function test_can_find_by_zip_code_via_api(): void
    {
        $response = $this->getJson('/api/thai-address/zip-code/10200');
        
        $response->assertStatus(200);
        $response->assertJsonStructure(['sub_district', 'district', 'province', 'zip_code']);
    }
    
    public function test_returns_404_for_invalid_zip_code(): void
    {
        $response = $this->getJson('/api/thai-address/zip-code/99999');
        
        $response->assertStatus(404);
    }
}
```

---

## 14. เผยแพร่ Package ขึ้น Packagist

```bash
# 1. สร้าง account บน packagist.org
# 2. Push code ขึ้น GitHub
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/yourname/thai-address.git
git push -u origin main

# 3. Tag version
git tag v1.0.0
git push --tags

# 4. Submit ที่ packagist.org > Submit > ใส่ GitHub URL

# 5. ตั้งค่า Auto-update ด้วย GitHub Webhook
# packagist.org จะให้ Webhook URL
# ใส่ที่ GitHub repo > Settings > Webhooks
```

---

## Quiz

**ข้อ 1:** `loadMigrationsFrom()` vs `publishes()` สำหรับ Migrations ต่างกันอย่างไร?

a) เหมือนกัน  
b) `loadMigrationsFrom()` รัน migration อัตโนมัติ, `publishes()` copy ไฟล์ไปยัง project  
c) `publishes()` รัน migration อัตโนมัติ  
d) `loadMigrationsFrom()` ไม่รองรับใน Production

**เฉลย:** b) `loadMigrationsFrom()` ทำให้ `php artisan migrate` รัน migrations จาก package โดยตรง ส่วน `publishes()` copy ไฟล์ migration ไปยัง project เพื่อให้ developer แก้ไขได้

---

**ข้อ 2:** เหตุใดจึงต้องใช้ `getTable()` ใน Package Models?

a) เพื่อ performance  
b) เพื่อให้ developer เปลี่ยนชื่อ table ผ่าน config ได้  
c) เพียงแค่ convention  
d) เพื่อ support MySQL เท่านั้น

**เฉลย:** b) การใช้ `getTable()` ที่ return ค่าจาก config ทำให้ developer สามารถเปลี่ยนชื่อ table ได้โดยไม่ต้อง override Model

---

**ข้อ 3:** `extra.laravel.providers` ใน composer.json ใช้ทำอะไร?

a) กำหนด dependencies  
b) Auto-discover Package เพื่อ register ServiceProvider อัตโนมัติ  
c) กำหนด PHP version  
d) ไม่มีประโยชน์

**เฉลย:** b) Laravel Package Discovery ใช้ข้อมูลนี้เพื่อ register ServiceProvider และ Aliases อัตโนมัติโดยไม่ต้องเพิ่มใน `config/app.php`

---

## สรุป

ใน Part นี้คุณได้เรียนรู้:
- **Package Structure:** โครงสร้าง Package ที่ดี
- **Service Provider:** register, boot, publishes, loadFrom
- **Facades:** สร้าง Static-like interface
- **Commands:** Artisan Commands ใน Package
- **Testing:** Unit และ Feature Tests ด้วย Orchestra Testbench
- **Workshop:** Package สำหรับ Thai Address

---

## ไปต่อ

➡️ [Part 047: Laravel Performance — Optimization เพื่อ Scale](./part-047-laravel-performance.md)
