# Part 097: PHP Internals & Advanced Performance

**ระดับ:** Expert  
**เวลาเรียน:** 6-8 ชั่วโมง  
**Prerequisites:** PHP Advanced OOP, C programming basics, Command line

---

## เป้าหมายของ Part นี้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
1. เข้าใจ Zend Engine architecture และ PHP execution pipeline
2. เข้าใจ Opcode และการทำงานของ OPcache
3. ใช้ JIT Compiler ใน PHP 8 เพื่อเพิ่มประสิทธิภาพ
4. เข้าใจ PHP Memory model และ Garbage Collection
5. รู้จักโครงสร้าง PHP Extension และเขียน extension ง่ายๆ ด้วย C

---

## 1. Zend Engine Architecture

### 1.1 ภาพรวม PHP Execution Pipeline

เมื่อ PHP ทำงาน มีขั้นตอนดังนี้:

```
PHP Source Code (.php)
        ↓
[Lexer/Tokenizer]
  - แยก source code เป็น tokens
  - เช่น T_ECHO, T_STRING, T_LNUMBER
        ↓
[Parser]
  - สร้าง Abstract Syntax Tree (AST)
  - ตรวจสอบ syntax
        ↓
[Compiler]
  - แปลง AST เป็น Opcodes
  - เก็บใน zend_op_array
        ↓
[OPcache]
  - Cache opcodes ใน shared memory
  - ข้ามขั้นตอน lexer/parser/compiler
        ↓
[Zend Virtual Machine]
  - Execute opcodes ทีละ instruction
  - จัดการ stack, heap, registers
        ↓
Output / Return Value
```

### 1.2 Tokenization

```php
<?php
$code = '<?php echo "Hello, " . $name; ?>';
$tokens = token_get_all($code);

foreach ($tokens as $token) {
  if (is_array($token)) {
    echo token_name($token[0]) . ': "' . $token[1] . '"' . PHP_EOL;
  } else {
    echo 'CHAR: "' . $token . '"' . PHP_EOL;
  }
}

/*
Output:
T_OPEN_TAG: "<?php "
T_ECHO: "echo"
T_WHITESPACE: " "
T_CONSTANT_ENCAPSED_STRING: ""Hello, ""
T_WHITESPACE: " "
CHAR: "."
T_WHITESPACE: " "
T_VARIABLE: "$name"
CHAR: ";"
T_WHITESPACE: " "
T_CLOSE_TAG: "?>"
*/
```

### 1.3 Abstract Syntax Tree (AST)

```php
<?php
// PHP 8 มี ast extension สำหรับดู AST
// ติดตั้ง: pecl install ast

$code = '<?php function add($a, $b) { return $a + $b; }';
$ast = \ast\parse_code($code, version: 80);
var_dump($ast);

/*
AST structure:
AST_STMT_LIST
  └── AST_FUNC_DECL (name: "add")
        ├── AST_PARAM_LIST
        │     ├── AST_PARAM ($a)
        │     └── AST_PARAM ($b)
        └── AST_STMT_LIST
              └── AST_RETURN
                    └── AST_BINARY_OP (+)
                          ├── AST_VAR ($a)
                          └── AST_VAR ($b)
*/
```

### 1.4 Zend Value (zval)

zval คือ data structure พื้นฐานที่ Zend Engine ใช้เก็บค่าทุกอย่างใน PHP

```c
/* Simplified zval structure (C) */
typedef struct _zval_struct {
    zend_value value;          // ค่าจริง (union)
    union {
        uint32_t type_info;
    } u1;
    union {
        uint32_t next;         // hash collision chain
        uint32_t cache_slot;
        uint32_t opline_num;
    } u2;
} zval;

typedef union _zend_value {
    zend_long   lval;          // int
    double      dval;          // float
    zend_refcounted *counted;  // string, array, object, resource
    zend_string *str;
    zend_array  *arr;
    zend_object *obj;
    zend_resource *res;
    zend_reference *ref;
} zend_value;
```

### 1.5 ประเภทข้อมูลและ Flags

```php
<?php
// ดูประเภทข้อมูลแบบ internal
$values = [
    42,          // IS_LONG
    3.14,        // IS_DOUBLE
    "hello",     // IS_STRING
    true,        // IS_TRUE
    false,       // IS_FALSE
    null,        // IS_NULL
    [],          // IS_ARRAY
    new stdClass, // IS_OBJECT
];

foreach ($values as $val) {
    echo gettype($val) . ': ' . var_export($val, true) . PHP_EOL;
}
```

---

## 2. Opcode & OPcache

### 2.1 Opcodes คืออะไร

Opcode (Operation Code) คือ instruction ระดับต่ำที่ Zend VM execute

```php
<?php
// ตัวอย่าง PHP code
function greet(string $name): string {
    return "Hello, " . $name . "!";
}
echo greet("World");
```

```bash
# ดู opcodes ด้วย VLD extension
php -d vld.active=1 -d vld.execute=0 file.php

# หรือ phpdbg
phpdbg -p file.php
```

```
# Opcodes ที่ได้ (approximation):
RECV $name
ROPE_INIT "Hello, "
ROPE_ADD $name
ROPE_END "!"
RETURN <result>
```

### 2.2 OPcache Configuration

```ini
; php.ini / opcache.ini

; เปิดใช้ OPcache
opcache.enable=1
opcache.enable_cli=0     ; สำหรับ CLI (ปกติ off)

; Memory
opcache.memory_consumption=256  ; MB
opcache.interned_strings_buffer=16  ; MB สำหรับ interned strings
opcache.max_accelerated_files=20000 ; จำนวน files สูงสุด

; Validation
opcache.validate_timestamps=0  ; Production: 0 (ปิด revalidation)
opcache.revalidate_freq=0      ; Production: 0

; OPcache file cache (เก็บลง disk ด้วย)
opcache.file_cache=/tmp/opcache
opcache.file_cache_only=0

; Optimization
opcache.optimization_level=0x7FFFBFFF  ; เปิดทุก optimization
opcache.opt_debug_level=0

; JIT (PHP 8+)
opcache.jit_buffer_size=100M
opcache.jit=1255  ; tracing JIT
```

### 2.3 ตรวจสอบ OPcache Status

```php
<?php
if (function_exists('opcache_get_status')) {
    $status = opcache_get_status(false);
    
    echo "OPcache Enabled: " . ($status['opcache_enabled'] ? 'Yes' : 'No') . PHP_EOL;
    echo "Cache Full: " . ($status['cache_full'] ? 'Yes' : 'No') . PHP_EOL;
    
    $mem = $status['memory_usage'];
    echo "Memory Used: " . round($mem['used_memory'] / 1024 / 1024, 2) . " MB" . PHP_EOL;
    echo "Memory Free: " . round($mem['free_memory'] / 1024 / 1024, 2) . " MB" . PHP_EOL;
    echo "Memory Wasted: " . round($mem['wasted_memory'] / 1024 / 1024, 2) . " MB" . PHP_EOL;
    
    $stats = $status['opcache_statistics'];
    echo "Cached Files: " . $stats['num_cached_scripts'] . PHP_EOL;
    echo "Hits: " . $stats['hits'] . PHP_EOL;
    echo "Misses: " . $stats['misses'] . PHP_EOL;
    $ratio = $stats['hits'] / max(1, $stats['hits'] + $stats['misses']) * 100;
    echo "Hit Rate: " . round($ratio, 2) . "%" . PHP_EOL;
}

// Reset OPcache (ใน development)
opcache_reset();

// Invalidate specific file
opcache_invalidate('/path/to/file.php', true);
```

### 2.4 Preloading (PHP 7.4+)

```php
<?php
// preload.php - โหลด classes ล่วงหน้า เมื่อ PHP เริ่มต้น

// ป้องกันการรันซ้ำ
if (PHP_SAPI !== 'fpm-fcgi') {
    return;
}

$files = [
    '/var/www/html/vendor/autoload.php',
    '/var/www/html/src/Models/User.php',
    '/var/www/html/src/Models/Post.php',
    '/var/www/html/src/Services/Database.php',
];

foreach ($files as $file) {
    opcache_compile_file($file);
}
```

```ini
; php.ini
opcache.preload=/var/www/html/preload.php
opcache.preload_user=www-data  ; user ที่รัน preload
```

---

## 3. JIT Compiler (PHP 8)

### 3.1 JIT คืออะไร

JIT (Just-In-Time) Compiler แปลง opcodes เป็น native machine code ขณะรัน แทนที่จะ interpret ทีละ opcode

```
ไม่มี JIT:
PHP Code → Opcodes → [Zend VM interprets each opcode] → Output

มี JIT:
PHP Code → Opcodes → [JIT compiles hot opcodes to machine code] → [CPU executes directly] → Output
```

### 3.2 JIT Configuration

```ini
; php.ini
; opcache.jit = CRTO format

; C = CPU-specific optimizations (0=disable, 1=enable)
; R = Register allocation (0=none, 1=local, 2=global)  
; T = JIT trigger (0=all, 1=functions, 2=hot, 3=tracing, 4=manual)
; O = Optimization level (0-5)

; Common values:
opcache.jit=1205  ; tracing JIT, basic
opcache.jit=1255  ; tracing JIT, aggressive (recommended)
opcache.jit=1235  ; function JIT
opcache.jit=off   ; ปิด JIT

opcache.jit_buffer_size=64M  ; memory สำหรับ JIT compiled code
```

### 3.3 เมื่อไร JIT มีประโยชน์

```php
<?php
// JIT ช่วยมาก: CPU-intensive computation
function mandelbrot(int $size): int {
    $count = 0;
    for ($y = 0; $y < $size; $y++) {
        for ($x = 0; $x < $size; $x++) {
            $cr = -2.0 + 3.0 * $x / $size;
            $ci = -1.5 + 3.0 * $y / $size;
            $zr = 0.0;
            $zi = 0.0;
            $i = 0;
            while ($zr * $zr + $zi * $zi < 4.0 && $i < 100) {
                [$zr, $zi] = [$zr * $zr - $zi * $zi + $cr, 2 * $zr * $zi + $ci];
                $i++;
            }
            if ($i === 100) $count++;
        }
    }
    return $count;
}

// Benchmark
$start = microtime(true);
$result = mandelbrot(500);
$end = microtime(true);

printf("Time: %.3f seconds, Count: %d\n", $end - $start, $result);
// ไม่มี JIT: ~0.8s, มี JIT: ~0.2s (4x faster!)
```

```php
// JIT ช่วยน้อย: I/O bound operations
function fetchData(): array {
    // Database queries, file I/O ไม่ได้รับประโยชน์จาก JIT
    return Database::query("SELECT * FROM users");
}
```

### 3.4 ตรวจสอบ JIT Status

```php
<?php
$status = opcache_get_status();
if (isset($status['jit'])) {
    $jit = $status['jit'];
    echo "JIT Enabled: " . ($jit['enabled'] ? 'Yes' : 'No') . PHP_EOL;
    echo "JIT Active: " . ($jit['on'] ? 'Yes' : 'No') . PHP_EOL;
    echo "Buffer Size: " . $jit['buffer_size'] . PHP_EOL;
    echo "Buffer Free: " . $jit['buffer_free'] . PHP_EOL;
}
```

---

## 4. PHP Memory Model & Garbage Collection

### 4.1 Memory Management

PHP ใช้ Memory Manager ในการจัดสรร memory:

```
Heap Memory (PHP Memory Pool)
├── Small blocks (< 3KB): พวงอยู่ใน free list
├── Large blocks (3KB - 2MB): จัดการแยก
└── Huge blocks (> 2MB): ใช้ mmap โดยตรง
```

### 4.2 Reference Counting

PHP ใช้ Reference Counting เป็น primary GC mechanism

```php
<?php
// แต่ละ value มี refcount
$a = "hello";     // refcount = 1
$b = $a;          // refcount = 2 (copy-on-write)
$c = &$a;         // refcount = 2 (actual reference)

unset($b);        // refcount = 1
unset($a);        // refcount = 1 ($c still holds)
unset($c);        // refcount = 0 → free memory

// ดู memory usage
echo memory_get_usage() . PHP_EOL;
echo memory_get_peak_usage() . PHP_EOL;
```

### 4.3 Copy-On-Write (COW)

```php
<?php
$a = range(1, 1000000);  // สร้าง array ใหญ่

$b = $a;  // ยังไม่ copy จริง (COW) - refcount เพิ่มแต่ pointer เดียวกัน
echo memory_get_usage() . PHP_EOL;  // ยังใช้ memory น้อย

$b[0] = 99;  // ตอนนี้ถึง copy จริง เพราะ modify
echo memory_get_usage() . PHP_EOL;  // memory เพิ่มขึ้น ~8MB

// ส่ง array ไป function โดยไม่ modify = ไม่ copy
function readArray(array $arr): int {
    return count($arr);  // ไม่ modify = COW ไม่ copy
}

// ส่ง by reference ถ้าต้องการหลีกเลี่ยง copy
function modifyArray(array &$arr): void {
    $arr[0] = 99;  // modify ของจริง ไม่ copy
}
```

### 4.4 Circular References และ Cycle Collector

```php
<?php
// Circular reference - refcount ไม่ถึง 0 แม้ไม่มี external reference
class Node {
    public ?Node $next = null;
    public string $data = '';
    
    public function __destruct() {
        echo "Destroying: {$this->data}\n";
    }
}

$node1 = new Node();
$node1->data = 'Node 1';
$node2 = new Node();
$node2->data = 'Node 2';

// สร้าง circular reference
$node1->next = $node2;
$node2->next = $node1;  // Circular!

// ลบ external references
unset($node1, $node2);
// refcount ไม่ถึง 0 เพราะ circular - รอ cycle collector!

// บังคับ run garbage collector
$collected = gc_collect_cycles();
echo "Collected: {$collected} cycles\n";
// Output: "Destroying: Node 1" และ "Destroying: Node 2"
```

### 4.5 Garbage Collector Configuration

```php
<?php
// ตรวจสอบ GC status
$status = gc_status();
echo "GC Enabled: " . ($status['running'] ? 'Running' : 'Not running') . PHP_EOL;
echo "GC Runs: " . $status['runs'] . PHP_EOL;
echo "Collected: " . $status['collected'] . PHP_EOL;
echo "Threshold: " . $status['threshold'] . PHP_EOL;
echo "Roots: " . $status['roots'] . PHP_EOL;

// ปิด/เปิด GC
gc_disable();
gc_enable();

// บังคับ collect
gc_collect_cycles();
```

```ini
; php.ini
gc_divisor = 1000       ; 1/1000 ของ gc_probability
gc_probability = 1      ; 0.1% chance ต่อ request
gc_maxlifetime = 1440   ; สำหรับ sessions เท่านั้น
```

### 4.6 Memory Leaks ใน PHP

```php
<?php
// Pattern ที่ทำให้ memory leak ใน long-running processes

// ❌ Bad: เก็บ closure ที่ capture large objects
class Processor {
    private array $handlers = [];
    
    public function addHandler(string $name, callable $handler): void {
        $this->handlers[$name] = $handler;
    }
}

$processor = new Processor();
$largeData = range(1, 1000000);

// Closure capture $largeData ทำให้ไม่ถูก free
$processor->addHandler('test', function() use ($largeData) {
    return count($largeData);
});

unset($largeData);  // $largeData ยังอยู่ใน closure!

// ✅ Good: pass data instead of capturing
$processor->addHandler('test', function(array $data) {
    return count($data);
});

// Memory monitoring
function memoryUsage(): string {
    return round(memory_get_usage(true) / 1024 / 1024, 2) . ' MB';
}

// ตรวจสอบ memory ใน loop
for ($i = 0; $i < 10000; $i++) {
    $obj = new SomeClass();
    // ถ้า memory เพิ่มขึ้นเรื่อยๆ = memory leak
    if ($i % 1000 === 0) {
        echo "Iteration {$i}: " . memoryUsage() . PHP_EOL;
        gc_collect_cycles();
    }
    unset($obj);
}
```

---

## 5. PHP Streams & Wrappers

### 5.1 PHP Streams

```php
<?php
// ประเภท Stream
// file://, http://, https://, ftp://, php://, data://

// อ่าน HTTP stream
$content = file_get_contents('https://api.example.com/data');

// Stream context
$context = stream_context_create([
    'http' => [
        'method' => 'POST',
        'header' => "Content-Type: application/json\r\n",
        'content' => json_encode(['key' => 'value']),
        'timeout' => 10,
    ],
]);

$response = file_get_contents('https://api.example.com/post', false, $context);

// PHP streams
$stdin = fopen('php://stdin', 'r');
$stdout = fopen('php://stdout', 'w');
$memory = fopen('php://memory', 'r+');  // ใช้ memory แทน disk
$temp = fopen('php://temp', 'r+');      // memory ถ้าน้อย, disk ถ้ามาก

// Stream filters
$handle = fopen('php://memory', 'r+');
stream_filter_append($handle, 'string.rot13');
fwrite($handle, "Hello World");
rewind($handle);
echo fread($handle, 100); // Uryyb Jbeyq (rot13)
fclose($handle);
```

### 5.2 Custom Stream Wrapper

```php
<?php
/**
 * Custom stream wrapper สำหรับ read/write encrypted files
 */
class EncryptedStreamWrapper {
    
    private $handle;
    private string $key = 'my-secret-key-32-bytes-long!!!!';
    
    /**
     * Register wrapper
     */
    public static function register(): void {
        stream_wrapper_register('encrypted', self::class);
    }
    
    public function stream_open(string $path, string $mode, int $options, ?string &$opened_path): bool {
        $realPath = str_replace('encrypted://', '', $path);
        $this->handle = fopen($realPath, $mode);
        return $this->handle !== false;
    }
    
    public function stream_write(string $data): int {
        $encrypted = openssl_encrypt($data, 'AES-256-CBC', $this->key, 0, substr($this->key, 0, 16));
        return fwrite($this->handle, $encrypted . "\n");
    }
    
    public function stream_read(int $count): string {
        $line = rtrim(fread($this->handle, $count));
        if (empty($line)) return '';
        return openssl_decrypt($line, 'AES-256-CBC', $this->key, 0, substr($this->key, 0, 16)) ?: '';
    }
    
    public function stream_eof(): bool {
        return feof($this->handle);
    }
    
    public function stream_close(): void {
        fclose($this->handle);
    }
    
    public function stream_stat(): array|false {
        return fstat($this->handle);
    }
}

// ลงทะเบียน wrapper
EncryptedStreamWrapper::register();

// ใช้งาน
file_put_contents('encrypted:///tmp/secret.enc', 'Sensitive data here');
$data = file_get_contents('encrypted:///tmp/secret.enc');
echo $data; // "Sensitive data here"
```

---

## 6. PHP Extension Basics (C)

### 6.1 โครงสร้าง PHP Extension

```c
/* hello.c - Simple PHP Extension */
#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include "php.h"
#include "php_ini.h"
#include "ext/standard/info.h"
#include "php_hello.h"

/* ประกาศ functions ของ extension */
PHP_FUNCTION(hello_world);
PHP_FUNCTION(add_numbers);

/* Function table */
static const zend_function_entry hello_functions[] = {
    PHP_FE(hello_world, NULL)
    PHP_FE(add_numbers, NULL)
    PHP_FE_END
};

/* Module entry */
zend_module_entry hello_module_entry = {
    STANDARD_MODULE_HEADER,
    "hello",           /* Extension name */
    hello_functions,   /* Functions */
    NULL,              /* MINIT */
    NULL,              /* MSHUTDOWN */
    NULL,              /* RINIT */
    NULL,              /* RSHUTDOWN */
    PHP_MINFO(hello),  /* MINFO */
    "1.0.0",           /* Version */
    STANDARD_MODULE_PROPERTIES
};

ZEND_GET_MODULE(hello)

/* PHP_MINFO */
PHP_MINFO_FUNCTION(hello) {
    php_info_print_table_start();
    php_info_print_table_header(2, "hello support", "enabled");
    php_info_print_table_row(2, "Version", "1.0.0");
    php_info_print_table_end();
}

/* hello_world() function */
PHP_FUNCTION(hello_world) {
    ZEND_PARSE_PARAMETERS_NONE();
    RETURN_STRING("Hello from C Extension!");
}

/* add_numbers(int $a, int $b) function */
PHP_FUNCTION(add_numbers) {
    zend_long a, b;
    
    ZEND_PARSE_PARAMETERS_START(2, 2)
        Z_PARAM_LONG(a)
        Z_PARAM_LONG(b)
    ZEND_PARSE_PARAMETERS_END();
    
    RETURN_LONG(a + b);
}
```

### 6.2 php_hello.h Header

```c
/* php_hello.h */
#ifndef PHP_HELLO_H
#define PHP_HELLO_H

extern zend_module_entry hello_module_entry;
#define phpext_hello_ptr &hello_module_entry

#define PHP_HELLO_VERSION "1.0.0"

PHP_MINFO_FUNCTION(hello);
PHP_FUNCTION(hello_world);
PHP_FUNCTION(add_numbers);

#endif /* PHP_HELLO_H */
```

### 6.3 config.m4 (สำหรับ compile)

```m4
# config.m4
PHP_ARG_ENABLE([hello],
  [whether to enable hello support],
  [AS_HELP_STRING([--enable-hello], [Enable hello support])])

if test "$PHP_HELLO" != "no"; then
  PHP_NEW_EXTENSION(hello, hello.c, $ext_shared)
fi
```

### 6.4 Compile และ Install Extension

```bash
# ติดตั้ง PHP development headers
sudo apt-get install php8.2-dev

# สร้างไฟล์ extension
mkdir /tmp/hello_ext
cd /tmp/hello_ext

# สร้างไฟล์ตามด้านบน (hello.c, php_hello.h, config.m4)

# Compile
phpize
./configure --enable-hello
make
make install

# หรือ install แบบ manual
cp modules/hello.so $(php-config --extension-dir)/

# เปิดใช้ใน php.ini
echo "extension=hello.so" >> $(php-config --ini-dir)/hello.ini

# ทดสอบ
php -r "echo hello_world(); echo add_numbers(3, 4);"
# Hello from C Extension!7
```

### 6.5 Extension ที่ซับซ้อนขึ้น: String Processing

```c
/* Function ที่ทำงานกับ PHP strings */
PHP_FUNCTION(my_strlen_utf8) {
    zend_string *str;
    
    ZEND_PARSE_PARAMETERS_START(1, 1)
        Z_PARAM_STR(str)
    ZEND_PARSE_PARAMETERS_END();
    
    /* นับ UTF-8 characters */
    size_t len = 0;
    const unsigned char *p = (unsigned char *)ZSTR_VAL(str);
    const unsigned char *end = p + ZSTR_LEN(str);
    
    while (p < end) {
        if (*p < 0x80) p += 1;         // 1-byte char
        else if (*p < 0xE0) p += 2;    // 2-byte char
        else if (*p < 0xF0) p += 3;    // 3-byte char
        else p += 4;                    // 4-byte char
        len++;
    }
    
    RETURN_LONG(len);
}

/* Function ที่ return array */
PHP_FUNCTION(get_system_info) {
    ZEND_PARSE_PARAMETERS_NONE();
    
    array_init(return_value);
    
    add_assoc_string(return_value, "php_version", PHP_VERSION);
    add_assoc_long(return_value, "memory_limit", PG(memory_limit));
    add_assoc_bool(return_value, "debug_build", ZEND_DEBUG);
}
```

---

## Workshop: PHP Extension ง่ายๆ ด้วย C

### เป้าหมาย
สร้าง PHP extension "fast_math" ที่มี functions:
- `fast_fibonacci(int $n)` - คำนวณ Fibonacci
- `fast_is_prime(int $n)` - ตรวจสอบจำนวนเฉพาะ
- `fast_gcd(int $a, int $b)` - หา GCD

### ขั้นตอนที่ 1: fast_math.c

```c
/* fast_math.c */
#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include "php.h"
#include "php_fast_math.h"

static const zend_function_entry fast_math_functions[] = {
    PHP_FE(fast_fibonacci, NULL)
    PHP_FE(fast_is_prime, NULL)
    PHP_FE(fast_gcd, NULL)
    PHP_FE_END
};

zend_module_entry fast_math_module_entry = {
    STANDARD_MODULE_HEADER,
    "fast_math",
    fast_math_functions,
    NULL, NULL, NULL, NULL,
    PHP_MINFO(fast_math),
    "1.0.0",
    STANDARD_MODULE_PROPERTIES
};

ZEND_GET_MODULE(fast_math)

PHP_MINFO_FUNCTION(fast_math) {
    php_info_print_table_start();
    php_info_print_table_header(2, "fast_math support", "enabled");
    php_info_print_table_end();
}

/* Fibonacci */
static zend_long fibonacci(zend_long n) {
    if (n <= 1) return n;
    zend_long a = 0, b = 1, temp;
    for (zend_long i = 2; i <= n; i++) {
        temp = a + b;
        a = b;
        b = temp;
    }
    return b;
}

PHP_FUNCTION(fast_fibonacci) {
    zend_long n;
    ZEND_PARSE_PARAMETERS_START(1, 1)
        Z_PARAM_LONG(n)
    ZEND_PARSE_PARAMETERS_END();
    
    if (n < 0) {
        zend_throw_exception(NULL, "n must be non-negative", 0);
        RETURN_THROWS();
    }
    
    RETURN_LONG(fibonacci(n));
}

/* Is Prime */
static int is_prime(zend_long n) {
    if (n < 2) return 0;
    if (n == 2) return 1;
    if (n % 2 == 0) return 0;
    for (zend_long i = 3; i * i <= n; i += 2) {
        if (n % i == 0) return 0;
    }
    return 1;
}

PHP_FUNCTION(fast_is_prime) {
    zend_long n;
    ZEND_PARSE_PARAMETERS_START(1, 1)
        Z_PARAM_LONG(n)
    ZEND_PARSE_PARAMETERS_END();
    
    RETURN_BOOL(is_prime(n));
}

/* GCD */
static zend_long gcd(zend_long a, zend_long b) {
    while (b != 0) {
        zend_long temp = b;
        b = a % b;
        a = temp;
    }
    return a;
}

PHP_FUNCTION(fast_gcd) {
    zend_long a, b;
    ZEND_PARSE_PARAMETERS_START(2, 2)
        Z_PARAM_LONG(a)
        Z_PARAM_LONG(b)
    ZEND_PARSE_PARAMETERS_END();
    
    RETURN_LONG(gcd(a, b));
}
```

### ขั้นตอนที่ 2: Compile

```bash
phpize
./configure --enable-fast_math
make && make install
echo "extension=fast_math.so" > /etc/php/8.2/cli/conf.d/fast_math.ini
```

### ขั้นตอนที่ 3: ทดสอบ

```php
<?php
// ทดสอบ extension
$start = microtime(true);
echo fast_fibonacci(50) . PHP_EOL;  // 12586269025
echo (microtime(true) - $start) . PHP_EOL;  // ~0.000001s (vs PHP ~0.001s)

var_dump(fast_is_prime(97));   // bool(true)
var_dump(fast_is_prime(100));  // bool(false)

echo fast_gcd(48, 18) . PHP_EOL;  // 6

// Benchmark vs PHP implementation
function phpFibonacci(int $n): int {
    if ($n <= 1) return $n;
    $a = 0; $b = 1;
    for ($i = 2; $i <= $n; $i++) {
        [$a, $b] = [$b, $a + $b];
    }
    return $b;
}

$iterations = 100000;
$start = microtime(true);
for ($i = 0; $i < $iterations; $i++) phpFibonacci(30);
$php_time = microtime(true) - $start;

$start = microtime(true);
for ($i = 0; $i < $iterations; $i++) fast_fibonacci(30);
$c_time = microtime(true) - $start;

printf("PHP: %.4fs, C Extension: %.4fs, Speedup: %.1fx\n",
    $php_time, $c_time, $php_time / $c_time);
```

---

## Quiz

**ข้อ 1:** PHP Opcodes คืออะไร?
- a) PHP source code ที่ minify แล้ว
- b) Intermediate instructions ที่ได้จาก compilation ของ PHP code ก่อนถูก execute โดย Zend VM
- c) Native machine code ที่ CPU รันโดยตรง
- d) Database query cache

**เฉลย:** b) Opcodes = intermediate representation ระหว่าง PHP source code และ machine code

---

**ข้อ 2:** PHP JIT Compiler ช่วยงานประเภทใดมากที่สุด?
- a) Database queries
- b) File I/O operations
- c) CPU-intensive mathematical computations
- d) HTTP requests

**เฉลย:** c) JIT ช่วย CPU-bound tasks เช่น math, algorithms แต่ไม่ช่วย I/O-bound tasks

---

**ข้อ 3:** Copy-On-Write (COW) ใน PHP ทำงานอย่างไร?
- a) Copy ทุกครั้งที่มีการ assignment
- b) ไม่มีการ copy เลย ใช้ pointer เสมอ
- c) Copy เฉพาะเมื่อมีการ modify ค่า ก่อนหน้านั้น share memory เดียวกัน
- d) Copy เฉพาะ arrays ไม่ copy strings

**เฉลย:** c) COW = copy ต่อเมื่อ write จริง ประหยัด memory ในกรณีที่อ่านอย่างเดียว

---

**ข้อ 4:** Circular Reference ทำให้เกิดปัญหาอะไรใน PHP?
- a) Infinite loop ทันที
- b) Reference counting ไม่สามารถ free memory ได้ ต้องใช้ cycle collector
- c) PHP crash
- d) ไม่มีปัญหา PHP จัดการได้เอง

**เฉลย:** b) Circular refs ทำให้ refcount ไม่ถึง 0 แม้ไม่มี external reference → cycle collector ต้องทำงาน

---

**ข้อ 5:** `opcache.validate_timestamps=0` ใน production ดีหรือไม่?
- a) ไม่ดี เพราะ PHP จะไม่อัปเดต code เมื่อมีการเปลี่ยนแปลง
- b) ดีมาก เพราะ PHP ไม่ต้อง check file timestamp ทุก request → เร็วขึ้น ต้อง clear opcache manual เมื่อ deploy
- c) ไม่มีความแตกต่าง
- d) ทำให้ JIT ทำงานไม่ได้

**เฉลย:** b) Production: ปิด validate_timestamps เพื่อ performance แล้ว clear OPcache เมื่อ deploy

---

## สรุป

ใน Part นี้คุณได้เรียนรู้:
- PHP Execution Pipeline: Lexer → Parser → Compiler → VM
- Opcodes และ OPcache configuration
- JIT Compiler และกรณีที่มีประโยชน์
- Memory model: Reference Counting, COW, Cycle Collector
- PHP Streams และ Custom Stream Wrappers
- การเขียน PHP Extension ด้วย C

**Part ถัดไป:** Part 098 - Advanced Testing (TDD, BDD, Mutation, Contract, Performance)
