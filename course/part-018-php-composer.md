# Part 018: PHP Composer ขั้นสูง

## ระดับ: Intermediate to Advanced
## เวลาเรียน: 3-4 ชั่วโมง

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ ผู้เรียนจะสามารถ:
- เข้าใจ composer.json ทุก field อย่างละเอียด
- เข้าใจ Semantic Versioning
- สร้าง PHP Package ของตัวเอง
- Submit package ไปยัง Packagist
- ตั้งค่า Private Package ด้วย Satis
- สร้างและ Publish PHP Package จริง

---

## 1. composer.json อย่างละเอียด

### โครงสร้าง composer.json แบบสมบูรณ์

```json
{
    "name": "vendor/package-name",
    "description": "คำอธิบาย package",
    "version": "1.2.3",
    "type": "library",
    "keywords": ["php", "library", "utility"],
    "homepage": "https://github.com/vendor/package",
    "readme": "README.md",
    "time": "2024-01-01",
    "license": "MIT",
    
    "authors": [
        {
            "name": "สมชาย PHP",
            "email": "somchai@example.com",
            "homepage": "https://somchai.dev",
            "role": "Developer"
        }
    ],
    
    "support": {
        "email": "support@example.com",
        "issues": "https://github.com/vendor/package/issues",
        "forum": "https://discourse.example.com",
        "wiki": "https://github.com/vendor/package/wiki",
        "source": "https://github.com/vendor/package"
    },
    
    "require": {
        "php": "^8.1",
        "ext-json": "*",
        "ext-mbstring": "*",
        "vendor/dependency": "^2.0"
    },
    
    "require-dev": {
        "phpunit/phpunit": "^10.0",
        "mockery/mockery": "^1.5",
        "phpstan/phpstan": "^1.0",
        "squizlabs/php_codesniffer": "^3.0"
    },
    
    "conflict": {
        "vendor/conflicting-package": ">=2.0"
    },
    
    "replace": {
        "vendor/old-package": "self.version"
    },
    
    "suggest": {
        "ext-redis": "สำหรับ cache ด้วย Redis",
        "vendor/optional-package": "เพิ่มฟีเจอร์ X"
    },
    
    "autoload": {
        "psr-4": {
            "Vendor\\Package\\": "src/"
        },
        "files": [
            "src/helpers.php"
        ]
    },
    
    "autoload-dev": {
        "psr-4": {
            "Vendor\\Package\\Tests\\": "tests/"
        }
    },
    
    "scripts": {
        "test": "phpunit",
        "test:coverage": "phpunit --coverage-html coverage",
        "lint": "phpcs src tests",
        "fix": "phpcbf src tests",
        "analyse": "phpstan analyse src",
        "check": [
            "@lint",
            "@analyse",
            "@test"
        ],
        "post-install-cmd": [
            "Vendor\\Package\\Scripts::postInstall"
        ],
        "post-update-cmd": [
            "@php artisan vendor:publish --force"
        ]
    },
    
    "extra": {
        "laravel": {
            "providers": [
                "Vendor\\Package\\ServiceProvider"
            ],
            "aliases": {
                "Package": "Vendor\\Package\\Facade"
            }
        },
        "branch-alias": {
            "dev-main": "2.0-dev"
        }
    },
    
    "config": {
        "preferred-install": "dist",
        "sort-packages": true,
        "optimize-autoloader": true,
        "platform": {
            "php": "8.1.0"
        },
        "allow-plugins": {
            "composer/package-versions-deprecated": true
        }
    },
    
    "minimum-stability": "stable",
    "prefer-stable": true
}
```

### อธิบาย Fields สำคัญ

#### name
```json
{
    "name": "vendor/package-name"
}
```
- รูปแบบ: `vendor/package` ตัวเล็กทั้งหมด
- ใช้ `-` แทน space
- Vendor = ชื่อ org/username ของคุณบน Packagist

#### type
```json
{
    "type": "library"
}
```
| Type | ความหมาย |
|------|---------|
| `library` | default, ใช้สำหรับ libraries (สร้าง dependencies) |
| `project` | สำหรับ projects/applications |
| `metapackage` | ไม่มีไฟล์, แค่ define dependencies |
| `composer-plugin` | plugin สำหรับ composer เอง |
| `symfony-bundle` | Symfony bundle |
| `wordpress-plugin` | WordPress plugin |

#### require และ Version Constraints

```json
{
    "require": {
        "php": "^8.1",
        "vendor/package": "^1.2.3",
        "vendor/other": "~2.1",
        "vendor/exact": "3.0.0",
        "vendor/range": ">=1.0 <2.0",
        "vendor/dev": "dev-main",
        "vendor/branch": "dev-feature-branch",
        "vendor/wildcard": "1.0.*"
    }
}
```

---

## 2. Semantic Versioning (SemVer)

### รูปแบบ: MAJOR.MINOR.PATCH

```
1.2.3
│ │ └── PATCH: bug fixes, ไม่ทำลาย API เดิม
│ └──── MINOR: features ใหม่, backward compatible
└────── MAJOR: breaking changes
```

### ตัวอย่าง SemVer

```
1.0.0  → Initial release
1.0.1  → แก้ bug
1.1.0  → เพิ่ม feature ใหม่ แต่ยังใช้ API เดิมได้
2.0.0  → เปลี่ยน API หลัก (breaking change)
```

### Version Constraints ใน Composer

```bash
# Exact version
composer require vendor/package:3.0.0

# Caret (^) - "compatible with"
# ^1.2.3 = >=1.2.3 <2.0.0
# ^0.3.0 = >=0.3.0 <0.4.0 (สำหรับ 0.x ระวัง minor!)
composer require vendor/package:^1.2.3

# Tilde (~) - "approximately"
# ~1.2.3 = >=1.2.3 <1.3.0
# ~1.2   = >=1.2.0 <2.0.0
composer require vendor/package:~1.2.3

# Wildcard (*)
# 1.0.*  = >=1.0.0 <1.1.0
composer require vendor/package:1.0.*

# Range
composer require "vendor/package:>=1.0 <2.0"

# Multiple constraints (OR)
composer require "vendor/package:^1.0 || ^2.0"

# Stability flags
composer require vendor/package:^2.0@beta
composer require vendor/package:dev-main
```

### เปรียบเทียบ ^ vs ~

```
^1.2.3 → >=1.2.3 <2.0.0    (นิยมกว่า)
~1.2.3 → >=1.2.3 <1.3.0    (strict กว่า)

^1.2   → >=1.2.0 <2.0.0
~1.2   → >=1.2.0 <2.0.0    (เหมือนกัน!)

^1     → >=1.0.0 <2.0.0
~1     → >=1.0.0 <2.0.0    (เหมือนกัน!)

^0.3   → >=0.3.0 <0.4.0    (พิเศษ: 0.x ไม่ข้าม minor)
~0.3   → >=0.3.0 <1.0.0
```

---

## 3. สร้าง PHP Package ของตัวเอง

### Workshop: สร้าง `phputils/string-helper`

#### โครงสร้าง Project

```
string-helper/
├── composer.json
├── README.md
├── LICENSE
├── src/
│   ├── StringHelper.php
│   ├── Inflector.php
│   └── Str.php (Facade/Helper)
├── tests/
│   ├── StringHelperTest.php
│   └── InflectorTest.php
└── .github/
    └── workflows/
        └── tests.yml
```

#### composer.json

```json
{
    "name": "phputils/string-helper",
    "description": "Useful string manipulation helpers for PHP",
    "version": "1.0.0",
    "type": "library",
    "keywords": ["string", "helper", "utility", "php"],
    "homepage": "https://github.com/phputils/string-helper",
    "license": "MIT",
    "authors": [
        {
            "name": "Your Name",
            "email": "your@email.com"
        }
    ],
    "require": {
        "php": "^8.1",
        "ext-mbstring": "*"
    },
    "require-dev": {
        "phpunit/phpunit": "^10.0"
    },
    "autoload": {
        "psr-4": {
            "PhpUtils\\StringHelper\\": "src/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "PhpUtils\\StringHelper\\Tests\\": "tests/"
        }
    },
    "scripts": {
        "test": "phpunit --testdox",
        "test:coverage": "phpunit --coverage-text"
    },
    "minimum-stability": "stable"
}
```

#### src/StringHelper.php

```php
<?php
namespace PhpUtils\StringHelper;

class StringHelper {
    /**
     * แปลง string เป็น camelCase
     */
    public static function toCamelCase(string $str): string {
        $str = preg_replace('/[^a-zA-Z0-9\s_-]/', '', $str);
        $str = str_replace(['-', '_'], ' ', $str);
        $str = ucwords(strtolower($str));
        $str = str_replace(' ', '', $str);
        return lcfirst($str);
    }
    
    /**
     * แปลง string เป็น snake_case
     */
    public static function toSnakeCase(string $str): string {
        $str = preg_replace('/([a-z])([A-Z])/', '$1_$2', $str);
        $str = preg_replace('/[-\s]+/', '_', $str);
        return strtolower(trim($str, '_'));
    }
    
    /**
     * แปลง string เป็น kebab-case
     */
    public static function toKebabCase(string $str): string {
        return str_replace('_', '-', static::toSnakeCase($str));
    }
    
    /**
     * แปลง string เป็น PascalCase
     */
    public static function toPascalCase(string $str): string {
        return ucfirst(static::toCamelCase($str));
    }
    
    /**
     * ตัด string ให้สั้นลง แต่ไม่ตัดกลางคำ
     */
    public static function truncate(
        string $str,
        int $length,
        string $suffix = '...'
    ): string {
        if (mb_strlen($str) <= $length) {
            return $str;
        }
        
        $truncated = mb_substr($str, 0, $length - mb_strlen($suffix));
        $lastSpace = mb_strrpos($truncated, ' ');
        
        if ($lastSpace !== false) {
            $truncated = mb_substr($truncated, 0, $lastSpace);
        }
        
        return $truncated . $suffix;
    }
    
    /**
     * Slug สำหรับ URL
     */
    public static function slug(string $str, string $separator = '-'): string {
        // แปลง Thai/Unicode characters
        $str = transliterator_transliterate('Any-Latin; Latin-ASCII', $str);
        $str = preg_replace('/[^a-zA-Z0-9\s]/', '', strtolower($str));
        $str = preg_replace('/[\s\-]+/', $separator, trim($str));
        return trim($str, $separator);
    }
    
    /**
     * ตรวจสอบว่า string เริ่มต้นด้วยข้อความที่กำหนด
     */
    public static function startsWith(string $str, string|array $needle): bool {
        foreach ((array) $needle as $n) {
            if (str_starts_with($str, $n)) {
                return true;
            }
        }
        return false;
    }
    
    /**
     * ตรวจสอบว่า string จบด้วยข้อความที่กำหนด
     */
    public static function endsWith(string $str, string|array $needle): bool {
        foreach ((array) $needle as $n) {
            if (str_ends_with($str, $n)) {
                return true;
            }
        }
        return false;
    }
    
    /**
     * นับจำนวนคำ (รองรับหลายภาษา)
     */
    public static function wordCount(string $str): int {
        return str_word_count(strip_tags($str));
    }
    
    /**
     * แทนที่ตัวแปรใน template
     */
    public static function template(string $template, array $vars): string {
        return preg_replace_callback(
            '/\{\{(\w+)\}\}/',
            fn($matches) => $vars[$matches[1]] ?? $matches[0],
            $template
        );
    }
    
    /**
     * สร้าง random string
     */
    public static function random(int $length = 16, string $charset = ''): string {
        $charset = $charset ?: 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
        $result = '';
        $max = strlen($charset) - 1;
        
        for ($i = 0; $i < $length; $i++) {
            $result .= $charset[random_int(0, $max)];
        }
        
        return $result;
    }
    
    /**
     * Mask ข้อมูล sensitive
     */
    public static function mask(string $str, int $start = 0, int $length = -1, string $char = '*'): string {
        $strLen = mb_strlen($str);
        $length = $length < 0 ? $strLen - $start : $length;
        
        return mb_substr($str, 0, $start)
            . str_repeat($char, $length)
            . mb_substr($str, $start + $length);
    }
    
    /**
     * ตรวจสอบว่าเป็น palindrome หรือไม่
     */
    public static function isPalindrome(string $str): bool {
        $cleaned = strtolower(preg_replace('/[^a-zA-Z0-9]/', '', $str));
        return $cleaned === strrev($cleaned);
    }
}
```

#### src/Inflector.php

```php
<?php
namespace PhpUtils\StringHelper;

class Inflector {
    private static array $pluralRules = [
        '/(quiz)$/i'               => '$1zes',
        '/^(oxen)$/i'              => '$1',
        '/^(ox)$/i'                => '$1en',
        '/([m|l])ice$/i'           => '$1ice',
        '/([m|l])ouse$/i'          => '$1ice',
        '/(pea)s$/i'               => '$1s',
        '/(pe)ople$/i'             => '$1ople',
        '/(child)ren$/i'           => '$1ren',
        '/(child)$/i'              => '$1ren',
        '/(x|ch|ss|sh)es$/i'       => '$1es',
        '/(x|ch|ss|sh)$/i'         => '$1es',
        '/([^aeiouy]|qu)ies$/i'    => '$1ies',
        '/([^aeiouy]|qu)y$/i'      => '$1ies',
        '/(hive)s$/i'              => '$1s',
        '/(hive)$/i'               => '$1s',
        '/([lr])ves$/i'            => '$1ves',
        '/([lr])f$/i'              => '$1ves',
        '/([^fo])ves$/i'           => '$1ves',
        '/([ti])a$/i'              => '$1a',
        '/((a)naly|(b)a|(d)iagno|(p)arenthe|(p)rogno|(s)ynop|(t)he)ses$/i' => '$1ses',
        '/((a)naly|(b)a|(d)iagno|(p)arenthe|(p)rogno|(s)ynop|(t)he)sis$/i' => '$1ses',
        '/([^s])s$/i'              => '$1s',
        '/s$/i'                    => 's',
        '/$/'                      => 's',
    ];
    
    private static array $irregulars = [
        'person' => 'people',
        'man'    => 'men',
        'child'  => 'children',
        'ox'     => 'oxen',
        'mouse'  => 'mice',
    ];
    
    private static array $uncountables = [
        'sheep', 'fish', 'deer', 'series', 'species',
        'money', 'rice', 'information', 'equipment',
    ];
    
    public static function pluralize(string $word): string {
        if (in_array(strtolower($word), self::$uncountables)) {
            return $word;
        }
        
        foreach (self::$irregulars as $singular => $plural) {
            if (strcasecmp($word, $singular) === 0) {
                return $plural;
            }
        }
        
        foreach (array_reverse(self::$pluralRules) as $pattern => $replacement) {
            if (preg_match($pattern, $word)) {
                return preg_replace($pattern, $replacement, $word);
            }
        }
        
        return $word . 's';
    }
    
    public static function tableize(string $className): string {
        return StringHelper::toSnakeCase(static::pluralize($className));
    }
    
    public static function classify(string $tableName): string {
        $singular = rtrim($tableName, 's');
        return StringHelper::toPascalCase($singular);
    }
}
```

#### tests/StringHelperTest.php

```php
<?php
namespace PhpUtils\StringHelper\Tests;

use PHPUnit\Framework\TestCase;
use PhpUtils\StringHelper\StringHelper;

class StringHelperTest extends TestCase {
    
    public function testToCamelCase(): void {
        $this->assertEquals('helloWorld', StringHelper::toCamelCase('hello world'));
        $this->assertEquals('helloWorld', StringHelper::toCamelCase('hello_world'));
        $this->assertEquals('helloWorld', StringHelper::toCamelCase('hello-world'));
        $this->assertEquals('helloWorldFoo', StringHelper::toCamelCase('hello-world-foo'));
    }
    
    public function testToSnakeCase(): void {
        $this->assertEquals('hello_world', StringHelper::toSnakeCase('HelloWorld'));
        $this->assertEquals('hello_world', StringHelper::toSnakeCase('helloWorld'));
        $this->assertEquals('hello_world', StringHelper::toSnakeCase('hello-world'));
    }
    
    public function testToKebabCase(): void {
        $this->assertEquals('hello-world', StringHelper::toKebabCase('helloWorld'));
        $this->assertEquals('hello-world-foo', StringHelper::toKebabCase('HelloWorldFoo'));
    }
    
    public function testTruncate(): void {
        $long = "สวัสดีชาวโลก นี่คือประโยคยาวๆ ที่เราต้องการตัด";
        $truncated = StringHelper::truncate($long, 15, '...');
        $this->assertLessThanOrEqual(15, strlen($truncated));
        $this->assertStringEndsWith('...', $truncated);
    }
    
    public function testMask(): void {
        $email = "user@example.com";
        $masked = StringHelper::mask($email, 4, 7);
        $this->assertEquals("user*******e.com", $masked);
        
        $phone = "0812345678";
        $masked = StringHelper::mask($phone, 3, 4);
        $this->assertEquals("081****678", $masked);
    }
    
    public function testRandom(): void {
        $r1 = StringHelper::random(16);
        $r2 = StringHelper::random(16);
        
        $this->assertEquals(16, strlen($r1));
        $this->assertNotEquals($r1, $r2); // ควรต่างกัน
    }
    
    public function testTemplate(): void {
        $template = "สวัสดี {{name}}! คุณมีอายุ {{age}} ปี";
        $result = StringHelper::template($template, ['name' => 'สมชาย', 'age' => '30']);
        $this->assertEquals("สวัสดี สมชาย! คุณมีอายุ 30 ปี", $result);
    }
    
    public function testIsPalindrome(): void {
        $this->assertTrue(StringHelper::isPalindrome("racecar"));
        $this->assertTrue(StringHelper::isPalindrome("A man a plan a canal Panama"));
        $this->assertFalse(StringHelper::isPalindrome("hello"));
    }
    
    /**
     * @dataProvider caseConversionProvider
     */
    public function testCaseConversions(string $input, string $expectedCamel, string $expectedSnake): void {
        $this->assertEquals($expectedCamel, StringHelper::toCamelCase($input));
        $this->assertEquals($expectedSnake, StringHelper::toSnakeCase($input));
    }
    
    public static function caseConversionProvider(): array {
        return [
            ['hello world', 'helloWorld', 'hello_world'],
            ['foo bar baz', 'fooBarBaz', 'foo_bar_baz'],
            ['PHP OOP', 'phpOop', 'php_oop'],
        ];
    }
}
```

---

## 4. Packagist Submission

### ขั้นตอนการ Submit Package

#### 1. เตรียม Repository บน GitHub

```bash
# สร้าง Git repository
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/username/string-helper.git
git push -u origin main
```

#### 2. สร้าง Tag สำหรับ Release

```bash
# สร้าง tag ตาม SemVer
git tag -a v1.0.0 -m "Release version 1.0.0"
git push origin v1.0.0

# สร้าง tag ใหม่หลัง update
git tag -a v1.0.1 -m "Fix bug in truncate method"
git push origin v1.0.1

git tag -a v1.1.0 -m "Add Inflector class"
git push origin v1.1.0
```

#### 3. Register บน Packagist

```
1. ไปที่ https://packagist.org
2. สมัครสมาชิก / Login
3. คลิก "Submit"
4. ใส่ URL GitHub Repository
5. คลิก "Check" เพื่อตรวจสอบ
6. คลิก "Submit" เพื่อ publish
```

#### 4. ตั้งค่า GitHub Webhook (Auto-Update)

```json
// เพิ่มใน GitHub Repository Settings > Webhooks
{
    "Payload URL": "https://packagist.org/api/github?username=packagist-username",
    "Content type": "application/json",
    "Secret": "<packagist-api-token>",
    "Events": ["Push"]
}
```

#### 5. ทดสอบการใช้งาน

```bash
# ใน project อื่น ทดสอบ install
composer require phputils/string-helper

# ถ้ายังไม่ได้ publish ทดสอบ local
composer require phputils/string-helper:@dev
```

---

## 5. Private Packages ด้วย Satis

Satis เป็น static Composer repository generator สำหรับ private packages

### ติดตั้ง Satis

```bash
composer create-project composer/satis --stability=dev --keep-vcs
cd satis
```

### satis.json Configuration

```json
{
    "name": "My Company Private Packages",
    "homepage": "https://packages.mycompany.com",
    "repositories": [
        {
            "type": "vcs",
            "url": "https://github.com/mycompany/private-package-1"
        },
        {
            "type": "vcs",
            "url": "https://bitbucket.org/mycompany/private-package-2"
        },
        {
            "type": "git",
            "url": "git@gitlab.com:mycompany/private-package-3.git"
        }
    ],
    "require-all": true,
    "archive": {
        "directory": "dist",
        "format": "tar",
        "skip-dev": true
    },
    "require-dependencies": true
}
```

### Build Satis Repository

```bash
# Build สร้างไฟล์ static HTML + packages.json
php bin/satis build satis.json web/

# Build เฉพาะ package ที่กำหนด
php bin/satis build satis.json web/ mycompany/private-package-1

# Output:
# web/
#   ├── index.html
#   ├── packages.json
#   └── dist/
#       └── mycompany/
#           └── private-package-1/
#               └── mycompany-private-package-1-1.0.0.tar
```

### ใช้งาน Private Repository

```json
// composer.json ของ project ที่ต้องการใช้ private package
{
    "repositories": [
        {
            "type": "composer",
            "url": "https://packages.mycompany.com"
        }
    ],
    "require": {
        "mycompany/private-package-1": "^1.0"
    },
    "config": {
        "http-basic": {
            "packages.mycompany.com": {
                "username": "deploy-user",
                "password": "deploy-token"
            }
        }
    }
}
```

### Docker Compose สำหรับ Satis Server

```yaml
# docker-compose.yml
version: '3.8'

services:
  satis:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./satis/web:/usr/share/nginx/html:ro
    
  satis-builder:
    image: composer:latest
    volumes:
      - .:/app
      - ~/.composer:/root/.composer
    command: >
      sh -c "
        cd /app/satis &&
        php bin/satis build satis.json web/ &&
        echo 'Build complete!'
      "
```

---

## 6. Workshop: สร้างและ Publish PHP Package

### โจทย์: สร้าง `myvendor/php-money` Package

สร้าง package สำหรับจัดการเงินและสกุลเงิน

#### โครงสร้างสมบูรณ์

```
php-money/
├── composer.json
├── README.md
├── CHANGELOG.md
├── LICENSE
├── src/
│   ├── Money.php
│   ├── Currency.php
│   ├── MoneyFormatter.php
│   ├── Exchange/
│   │   ├── ExchangeRateInterface.php
│   │   └── StaticExchangeRate.php
│   └── Exceptions/
│       ├── InvalidMoneyException.php
│       └── CurrencyMismatchException.php
└── tests/
    ├── MoneyTest.php
    └── MoneyFormatterTest.php
```

#### src/Currency.php

```php
<?php
namespace MyVendor\Money;

use MyVendor\Money\Exceptions\InvalidMoneyException;

final class Currency {
    private static array $currencies = [
        'THB' => ['name' => 'Thai Baht', 'symbol' => '฿', 'decimals' => 2],
        'USD' => ['name' => 'US Dollar', 'symbol' => '$', 'decimals' => 2],
        'EUR' => ['name' => 'Euro', 'symbol' => '€', 'decimals' => 2],
        'JPY' => ['name' => 'Japanese Yen', 'symbol' => '¥', 'decimals' => 0],
        'GBP' => ['name' => 'British Pound', 'symbol' => '£', 'decimals' => 2],
        'BTC' => ['name' => 'Bitcoin', 'symbol' => '₿', 'decimals' => 8],
    ];
    
    private string $code;
    
    public function __construct(string $code) {
        $code = strtoupper($code);
        if (!isset(self::$currencies[$code])) {
            throw new InvalidMoneyException("Unknown currency: {$code}");
        }
        $this->code = $code;
    }
    
    public function getCode(): string {
        return $this->code;
    }
    
    public function getName(): string {
        return self::$currencies[$this->code]['name'];
    }
    
    public function getSymbol(): string {
        return self::$currencies[$this->code]['symbol'];
    }
    
    public function getDecimals(): int {
        return self::$currencies[$this->code]['decimals'];
    }
    
    public function equals(Currency $other): bool {
        return $this->code === $other->code;
    }
    
    public function __toString(): string {
        return $this->code;
    }
    
    public static function THB(): self { return new self('THB'); }
    public static function USD(): self { return new self('USD'); }
    public static function EUR(): self { return new self('EUR'); }
    public static function JPY(): self { return new self('JPY'); }
}
```

#### src/Money.php

```php
<?php
namespace MyVendor\Money;

use MyVendor\Money\Exceptions\CurrencyMismatchException;
use MyVendor\Money\Exceptions\InvalidMoneyException;

final class Money {
    private int $amount; // เก็บเป็น smallest unit (สตางค์ สำหรับ THB)
    
    public function __construct(
        int|float $amount,
        private Currency $currency
    ) {
        // แปลงเป็น integer (หน่วยย่อยสุด)
        $multiplier = 10 ** $currency->getDecimals();
        $this->amount = (int) round($amount * $multiplier);
    }
    
    public static function of(int|float $amount, string $currency): self {
        return new self($amount, new Currency($currency));
    }
    
    public static function THB(int|float $amount): self {
        return new self($amount, Currency::THB());
    }
    
    public static function USD(int|float $amount): self {
        return new self($amount, Currency::USD());
    }
    
    public function getAmount(): float {
        $divisor = 10 ** $this->currency->getDecimals();
        return $this->amount / $divisor;
    }
    
    public function getAmountInSmallestUnit(): int {
        return $this->amount;
    }
    
    public function getCurrency(): Currency {
        return $this->currency;
    }
    
    private function assertSameCurrency(Money $other): void {
        if (!$this->currency->equals($other->currency)) {
            throw new CurrencyMismatchException(
                "Cannot operate on {$this->currency} and {$other->currency}"
            );
        }
    }
    
    public function add(Money $other): self {
        $this->assertSameCurrency($other);
        $result = clone $this;
        $result->amount = $this->amount + $other->amount;
        return $result;
    }
    
    public function subtract(Money $other): self {
        $this->assertSameCurrency($other);
        $result = clone $this;
        $result->amount = $this->amount - $other->amount;
        return $result;
    }
    
    public function multiply(float $factor): self {
        $result = clone $this;
        $result->amount = (int) round($this->amount * $factor);
        return $result;
    }
    
    public function divide(float $divisor): self {
        if ($divisor == 0) {
            throw new InvalidMoneyException("Cannot divide by zero");
        }
        $result = clone $this;
        $result->amount = (int) round($this->amount / $divisor);
        return $result;
    }
    
    public function equals(Money $other): bool {
        return $this->amount === $other->amount
            && $this->currency->equals($other->currency);
    }
    
    public function isGreaterThan(Money $other): bool {
        $this->assertSameCurrency($other);
        return $this->amount > $other->amount;
    }
    
    public function isLessThan(Money $other): bool {
        $this->assertSameCurrency($other);
        return $this->amount < $other->amount;
    }
    
    public function isZero(): bool {
        return $this->amount === 0;
    }
    
    public function isPositive(): bool {
        return $this->amount > 0;
    }
    
    public function isNegative(): bool {
        return $this->amount < 0;
    }
    
    /**
     * แบ่งเงินเป็นส่วนๆ (ไม่สูญเสียเศษสตางค์)
     */
    public function allocate(array $ratios): array {
        $total = array_sum($ratios);
        $results = [];
        $remainder = $this->amount;
        
        foreach ($ratios as $ratio) {
            $share = (int) floor($this->amount * $ratio / $total);
            $results[] = $share;
            $remainder -= $share;
        }
        
        // กระจายเศษที่เหลือ
        for ($i = 0; $i < $remainder; $i++) {
            $results[$i]++;
        }
        
        return array_map(function($amount) {
            $money = clone $this;
            $money->amount = $amount;
            return $money;
        }, $results);
    }
    
    public function __toString(): string {
        $symbol = $this->currency->getSymbol();
        $amount = number_format($this->getAmount(), $this->currency->getDecimals());
        return "{$symbol}{$amount}";
    }
}
```

#### src/MoneyFormatter.php

```php
<?php
namespace MyVendor\Money;

class MoneyFormatter {
    public function format(Money $money, string $locale = 'th_TH'): string {
        $amount = $money->getAmount();
        $currency = $money->getCurrency();
        
        // ตัวอย่าง formatting สำหรับ locale ต่างๆ
        return match($locale) {
            'th_TH' => $currency->getSymbol() . number_format($amount, $currency->getDecimals()),
            'en_US' => number_format($amount, $currency->getDecimals()) . ' ' . $currency->getCode(),
            'de_DE' => number_format($amount, $currency->getDecimals(), ',', '.') . ' ' . $currency->getSymbol(),
            default => (string) $money,
        };
    }
    
    public function formatThaiWords(Money $money): string {
        if (!$money->getCurrency()->equals(Currency::THB())) {
            return (string) $money;
        }
        
        $amount = $money->getAmount();
        $baht = (int) $amount;
        $satang = (int) round(($amount - $baht) * 100);
        
        $bahtText = $this->numberToThaiWords($baht) . "บาท";
        $satangText = $satang > 0 ? $this->numberToThaiWords($satang) . "สตางค์" : "ถ้วน";
        
        return $bahtText . $satangText;
    }
    
    private function numberToThaiWords(int $number): string {
        $ones = ['', 'หนึ่ง', 'สอง', 'สาม', 'สี่', 'ห้า', 'หก', 'เจ็ด', 'แปด', 'เก้า'];
        $tens = ['', 'สิบ', 'ยี่สิบ', 'สามสิบ', 'สี่สิบ', 'ห้าสิบ', 'หกสิบ', 'เจ็ดสิบ', 'แปดสิบ', 'เก้าสิบ'];
        
        if ($number < 10) return $ones[$number];
        if ($number < 100) {
            $ten = intdiv($number, 10);
            $one = $number % 10;
            return $tens[$ten] . ($one === 1 && $ten > 1 ? 'เอ็ด' : $ones[$one]);
        }
        if ($number < 1000) {
            $h = intdiv($number, 100);
            return $ones[$h] . 'ร้อย' . $this->numberToThaiWords($number % 100);
        }
        
        return (string) $number; // simplified for large numbers
    }
}
```

#### tests/MoneyTest.php

```php
<?php
namespace MyVendor\Money\Tests;

use PHPUnit\Framework\TestCase;
use MyVendor\Money\Money;
use MyVendor\Money\Currency;
use MyVendor\Money\Exceptions\CurrencyMismatchException;

class MoneyTest extends TestCase {
    
    public function testCreateMoney(): void {
        $money = Money::THB(100.50);
        $this->assertEquals(100.50, $money->getAmount());
        $this->assertEquals('THB', (string) $money->getCurrency());
    }
    
    public function testAddMoney(): void {
        $m1 = Money::THB(100);
        $m2 = Money::THB(50.50);
        $result = $m1->add($m2);
        
        $this->assertEquals(150.50, $result->getAmount());
    }
    
    public function testSubtractMoney(): void {
        $m1 = Money::THB(100);
        $m2 = Money::THB(30);
        $result = $m1->subtract($m2);
        
        $this->assertEquals(70, $result->getAmount());
    }
    
    public function testMultiply(): void {
        $m = Money::THB(100);
        $result = $m->multiply(1.5);
        $this->assertEquals(150, $result->getAmount());
    }
    
    public function testCurrencyMismatchThrows(): void {
        $thb = Money::THB(100);
        $usd = Money::USD(100);
        
        $this->expectException(CurrencyMismatchException::class);
        $thb->add($usd);
    }
    
    public function testAllocate(): void {
        $money = Money::THB(100);
        [$share1, $share2, $share3] = $money->allocate([1, 1, 1]);
        
        // แต่ละส่วนได้ประมาณ 33.33
        $total = $share1->add($share2)->add($share3);
        $this->assertEquals(100, $total->getAmount());
    }
    
    public function testImmutability(): void {
        $original = Money::THB(100);
        $result = $original->add(Money::THB(50));
        
        // original ต้องไม่เปลี่ยน
        $this->assertEquals(100, $original->getAmount());
        $this->assertEquals(150, $result->getAmount());
    }
    
    public function testComparison(): void {
        $m1 = Money::THB(100);
        $m2 = Money::THB(200);
        
        $this->assertTrue($m2->isGreaterThan($m1));
        $this->assertTrue($m1->isLessThan($m2));
        $this->assertFalse($m1->equals($m2));
        $this->assertTrue($m1->equals(Money::THB(100)));
    }
    
    public function testToString(): void {
        $m = Money::THB(1234.56);
        $this->assertStringContainsString('1,234.56', (string) $m);
    }
}
```

---

## Quiz

### คำถาม 1
`^2.1.3` หมายความว่าอะไร?
- A. >=2.1.3 <3.0.0
- B. >=2.1.3 <2.2.0
- C. =2.1.3 เท่านั้น
- D. >=2.0.0 <3.0.0

**เฉลย: A** - Caret constraint อนุญาต minor และ patch updates แต่ไม่ข้าม major version

### คำถาม 2
ต้องการให้ script รันหลัง `composer install` โดยอัตโนมัติ ต้องใช้ field อะไร?
- A. `scripts > post-install-cmd`
- B. `scripts > install`
- C. `extra > install-script`
- D. `config > post-install`

**เฉลย: A** - `post-install-cmd` รันหลัง `composer install` สำเร็จ

### คำถาม 3
ความแตกต่างระหว่าง `require` และ `require-dev` คืออะไร?

**เฉลย:** 
- `require`: dependencies ที่จำเป็นทั้งใน production และ development
- `require-dev`: dependencies สำหรับ development เท่านั้น (tests, code analysis tools)
- `composer install --no-dev` จะข้าม `require-dev` packages

### คำถาม 4
`"type": "library"` vs `"type": "project"` ต่างกันอย่างไร?

**เฉลย:**
- `library`: สร้างสำหรับให้คนอื่น depend on, Packagist จะ index
- `project`: ตัว application เอง, ไม่ควร publish ขึ้น Packagist

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- **composer.json fields** ทั้งหมดและความหมาย
- **Semantic Versioning** และ version constraints (^, ~, *, range)
- **การสร้าง Package** พร้อม autoloading และ tests
- **Packagist** การ submit และ publish package
- **Private Packages** ด้วย Satis สำหรับ internal use
- **Workshop** สร้าง Money library ที่ใช้งานได้จริง

---

## ➡️ Part ถัดไป

[Part 019: PHP Error Handling และ Logging](./part-019-php-error-handling.md)
