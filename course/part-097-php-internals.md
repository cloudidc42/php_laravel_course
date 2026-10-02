# Part 97: PHP Internals

## บทนำ

การเข้าใจ PHP Internals ช่วยให้เราเขียน Code ที่มีประสิทธิภาพมากขึ้น และแก้ปัญหาที่ซับซ้อนได้ บทนี้จะครอบคลุม:
- Zend Engine Architecture
- PHP Extension Development (C)
- JIT Compiler
- Memory Management
- Garbage Collection

---

## Zend Engine

Zend Engine คือ Core Engine ของ PHP ทำหน้าที่ Compile และ Execute PHP Code

### การทำงานของ PHP

```
PHP Source Code
      ↓
[Lexer/Tokenizer]  ← แปลง Source เป็น Tokens
      ↓
[Parser]           ← สร้าง Abstract Syntax Tree (AST)
      ↓
[Compiler]         ← แปลง AST เป็น Opcodes
      ↓
[OPcache]          ← Cache Opcodes (ไม่ต้อง Compile ซ้ำ)
      ↓
[Zend VM]          ← Execute Opcodes
      ↓
Output
```

### PHP Opcodes

```php
<?php

// ดู Opcodes ของ PHP Code
// php -r "opcache_compile_file('test.php'); var_dump(opcache_get_status());"

// หรือใช้ php -d opcache.opt_debug_level=0x10000 test.php

// Example PHP Code
function add(int $a, int $b): int
{
    return $a + $b;
}

$result = add(3, 4);
echo $result;

/*
Opcodes (ที่ Zend VM Execute):
0000 INIT_FCALL 'add'
0001 SEND_VAL 3
0002 SEND_VAL 4
0003 DO_FCALL
0004 ASSIGN $result
0005 ECHO $result
0006 RETURN 1
*/
```

### Variable Storage (zval)

```c
// ทุก PHP Variable เก็บเป็น zval struct ใน C
typedef struct _zval_struct {
    zend_value value;        // ค่าจริงๆ
    union {
        uint32_t type_info;
        struct {
            ZEND_ENDIAN_LOHI_3(
                zend_uchar    type,        // IS_NULL, IS_BOOL, IS_LONG, etc.
                zend_uchar    type_flags,
                union {
                    uint16_t  extra;
                }
            )
        } v;
    } u1;
    union {
        uint32_t     next;   // สำหรับ hash collision chain
        uint32_t     cache_slot;
        uint32_t     opline_num;
        uint32_t     lineno;
        uint32_t     num_args;
        uint32_t     fe_pos;
        uint32_t     fe_iter_idx;
        uint32_t     property_guard;
        uint32_t     constant_flags;
        uint32_t     extra;
    } u2;
} zval;

typedef union _zend_value {
    zend_long         lval;   // long integer value
    double            dval;   // double floating point value
    zend_refcounted  *counted;
    zend_string      *str;    // string
    zend_array       *arr;    // array
    zend_object      *obj;    // object
    zend_resource    *res;    // resource
    zend_reference   *ref;    // reference
    zend_ast_ref     *ast;    // AST
    zval             *zv;
    void             *ptr;
    zend_class_entry *ce;
    zend_function    *func;
    struct {
        uint32_t w1;
        uint32_t w2;
    } ww;
} zend_value;
```

---

## PHP Extension Development (C)

### สร้าง Extension อย่างง่าย

```c
/* hello_extension/hello.c */

#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include "php.h"
#include "php_ini.h"
#include "ext/standard/info.h"

/* Function Declarations */
PHP_FUNCTION(hello_world);
PHP_FUNCTION(calculate_fibonacci);

/* Module entry */
zend_module_entry hello_module_entry = {
    STANDARD_MODULE_HEADER,
    "hello",              /* Extension name */
    hello_functions,      /* Function list */
    PHP_MINIT(hello),    /* Module init */
    PHP_MSHUTDOWN(hello),/* Module shutdown */
    PHP_RINIT(hello),    /* Request init */
    PHP_RSHUTDOWN(hello),/* Request shutdown */
    PHP_MINFO(hello),    /* Module info */
    "1.0",               /* Extension version */
    STANDARD_MODULE_PROPERTIES
};

/* Function Entries */
static const zend_function_entry hello_functions[] = {
    PHP_FE(hello_world, NULL)
    PHP_FE(calculate_fibonacci, arginfo_calculate_fibonacci)
    PHP_FE_END
};

/* Module Init */
PHP_MINIT_FUNCTION(hello)
{
    /* Register constants */
    REGISTER_LONG_CONSTANT("HELLO_VERSION", 1, CONST_CS | CONST_PERSISTENT);
    return SUCCESS;
}

PHP_MSHUTDOWN_FUNCTION(hello)
{
    return SUCCESS;
}

PHP_RINIT_FUNCTION(hello)
{
    return SUCCESS;
}

PHP_RSHUTDOWN_FUNCTION(hello)
{
    return SUCCESS;
}

PHP_MINFO_FUNCTION(hello)
{
    php_info_print_table_start();
    php_info_print_table_row(2, "Hello Extension", "Enabled");
    php_info_print_table_row(2, "Version", "1.0");
    php_info_print_table_end();
}

/* hello_world() implementation */
PHP_FUNCTION(hello_world)
{
    zend_string *name;
    
    /* Parse parameters: s = string */
    if (zend_parse_parameters(ZEND_NUM_ARGS(), "S", &name) == FAILURE) {
        RETURN_FALSE;
    }
    
    /* Build return string */
    smart_str output = {0};
    smart_str_appends(&output, "Hello, ");
    smart_str_append(&output, name);
    smart_str_appendc(&output, '!');
    smart_str_0(&output);
    
    RETURN_STR(output.s);
}

/* Fibonacci implementation */
static zend_long fibonacci(zend_long n)
{
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

/* Argument info for calculate_fibonacci */
ZEND_BEGIN_ARG_WITH_RETURN_TYPE_INFO_EX(arginfo_calculate_fibonacci, 0, 1, IS_LONG, 0)
    ZEND_ARG_TYPE_INFO(0, n, IS_LONG, 0)
ZEND_END_ARG_INFO()

PHP_FUNCTION(calculate_fibonacci)
{
    zend_long n;
    
    if (zend_parse_parameters(ZEND_NUM_ARGS(), "l", &n) == FAILURE) {
        RETURN_FALSE;
    }
    
    if (n < 0 || n > 90) {
        zend_throw_exception(
            zend_ce_invalid_argument_exception,
            "n must be between 0 and 90",
            0
        );
        RETURN_THROWS();
    }
    
    RETURN_LONG(fibonacci(n));
}
```

```c
/* hello_extension/php_hello.h */

#ifndef PHP_HELLO_H
#define PHP_HELLO_H

extern zend_module_entry hello_module_entry;
#define phpext_hello_ptr &hello_module_entry

#define PHP_HELLO_VERSION "1.0"

PHP_MINIT_FUNCTION(hello);
PHP_MSHUTDOWN_FUNCTION(hello);
PHP_RINIT_FUNCTION(hello);
PHP_RSHUTDOWN_FUNCTION(hello);
PHP_MINFO_FUNCTION(hello);

PHP_FUNCTION(hello_world);
PHP_FUNCTION(calculate_fibonacci);

#endif /* PHP_HELLO_H */
```

```xml
<!-- hello_extension/config.m4 -->
PHP_ARG_ENABLE(hello, whether to enable hello support,
[  --enable-hello          Enable hello support])

if test "$PHP_HELLO" != "no"; then
    PHP_NEW_EXTENSION(hello, hello.c, $ext_shared)
fi
```

```bash
# Build Extension
cd hello_extension
phpize
./configure --enable-hello
make
make install

# เพิ่มใน php.ini
extension=hello.so

# ทดสอบ
php -r "echo hello_world('PHP Developer');"
# Hello, PHP Developer!

php -r "echo calculate_fibonacci(10);"
# 55
```

---

## JIT Compiler

PHP 8.0+ มี JIT (Just-In-Time) Compiler ที่แปลง Opcodes เป็น Native Machine Code

### การทำงานของ JIT

```
PHP Opcodes
      ↓
[JIT Profiler]     ← เก็บ Statistics ว่า Code ไหนถูก Execute บ่อย
      ↓
[JIT Compiler]     ← แปลง Hot Code เป็น Machine Code
      ↓
Native Machine Code ← เร็วกว่า Opcode Execution
```

### JIT Configuration

```ini
; php.ini
[JIT]
opcache.jit=tracing          ; tracing = ดีที่สุดสำหรับ General Purpose
                              ; function = ดีสำหรับ OOP
                              ; disable = ปิด JIT
opcache.jit_buffer_size=256M  ; ขนาด Buffer สำหรับ Native Code

; JIT Optimization Levels:
; 0 = ปิด JIT
; 1 = JIT function อย่างง่าย
; 2 = JIT with optimization
; 3 = รวม Register Allocation
; 4 = ทั้งหมด
opcache.jit_opt_level=4
```

### เมื่อไหร่ JIT ช่วยได้

```php
<?php

// JIT ช่วยมากสำหรับ CPU-bound tasks
// เช่น Mathematical Computations

// ตัวอย่าง: Mandelbrot Set
function mandelbrot(float $cx, float $cy, int $maxIter): int
{
    $x = $cx;
    $y = $cy;
    
    for ($i = 0; $i < $maxIter; $i++) {
        $x2 = $x * $x;
        $y2 = $y * $y;
        
        if ($x2 + $y2 > 4.0) {
            return $i;
        }
        
        $y = 2 * $x * $y + $cy;
        $x = $x2 - $y2 + $cx;
    }
    
    return $maxIter;
}

// ทดสอบ Performance
$start = microtime(true);

for ($x = -2; $x < 1; $x += 0.01) {
    for ($y = -1.5; $y < 1.5; $y += 0.01) {
        mandelbrot($x, $y, 100);
    }
}

$elapsed = microtime(true) - $start;
echo "Time: {$elapsed}s\n";

// ไม่มี JIT: ~2.5s
// มี JIT (tracing): ~0.8s (เร็วขึ้น 3x)

// JIT ไม่ช่วยมากนักสำหรับ I/O-bound tasks
// เช่น Web Applications ที่รอ DB Query
```

---

## Memory Management

### PHP Memory Model

```php
<?php

// ดู Memory Usage
echo memory_get_usage() . " bytes\n";
echo memory_get_peak_usage() . " bytes\n";

// Limit
echo ini_get('memory_limit') . "\n"; // 128M default

// Reference Counting
$a = "Hello"; // refcount = 1
$b = $a;      // refcount = 2, Copy-on-Write
$b .= " World"; // refcount แยกออกจากกัน (COW triggered)

// ตรวจสอบ refcount
$a = "Hello";
$b = $a;
var_dump(debug_zval_refcount($a)); // 2

// การจัดการ Memory Leak
function processLargeFile(string $path): void
{
    $handle = fopen($path, 'r');
    
    // อ่านทีละ 4KB ไม่ Load ทั้ง File
    while (!feof($handle)) {
        $chunk = fread($handle, 4096);
        process($chunk);
        unset($chunk); // Free memory
    }
    
    fclose($handle);
}

// Generator สำหรับ Large Data
function readLines(string $filename): \Generator
{
    $handle = fopen($filename, 'r');
    
    while ($line = fgets($handle)) {
        yield $line;
    }
    
    fclose($handle);
}

// ใช้ Memory น้อยมาก
foreach (readLines('/var/log/nginx/access.log') as $line) {
    processLine($line);
}

// Copy-on-Write
$original = range(1, 10000); // Allocated once
$copy = $original; // ยังไม่ Copy จริงๆ

// เมื่อแก้ไข copy เท่านั้น ถึงจะ Copy
$copy[] = 10001; // Copy triggered
```

### Weak References

```php
<?php

// Weak Reference ไม่ป้องกัน Garbage Collection
class ExpensiveObject
{
    public string $data;
    
    public function __construct(string $data)
    {
        $this->data = $data;
        echo "Created ExpensiveObject\n";
    }
    
    public function __destruct()
    {
        echo "Destroyed ExpensiveObject\n";
    }
}

$obj = new ExpensiveObject("important data");
$weakRef = WeakReference::create($obj);

echo $weakRef->get()?->data . "\n"; // "important data"

unset($obj); // ทำลาย Object

echo var_export($weakRef->get(), true) . "\n"; // NULL - Object ถูกทำลายแล้ว

// WeakMap (PHP 8.0+) - สำหรับ Caching ที่ไม่ป้องกัน GC
$cache = new \WeakMap();

$obj1 = new \stdClass();
$cache[$obj1] = 'cached value';

echo $cache[$obj1] . "\n"; // "cached value"

unset($obj1); // ลบทั้ง Object และ Cache entry
```

---

## Garbage Collection

### Reference Counting + Cycle Collector

```php
<?php

// Reference Counting - ลบทันทีเมื่อ refcount = 0
$a = new \stdClass();  // refcount: 1
$b = $a;               // refcount: 2
unset($a);             // refcount: 1 (ยังไม่ลบ)
unset($b);             // refcount: 0 (ลบทันที!)

// Circular Reference - ต้องการ Cycle Collector
class Node
{
    public ?Node $next = null;
    
    public function __destruct()
    {
        echo "Node destroyed\n";
    }
}

$node1 = new Node();
$node2 = new Node();

// สร้าง Circular Reference
$node1->next = $node2;
$node2->next = $node1;

unset($node1);
unset($node2);

// ไม่ถูกลบทันที! เพราะ refcount ยังไม่เป็น 0
// PHP จะ GC ทีหลัง เมื่อ cycle buffer เต็ม หรือเรียก gc_collect_cycles()

gc_collect_cycles(); // Force garbage collection
// "Node destroyed" x2

// ดู GC Stats
$stats = gc_status();
echo "GC runs: {$stats['runs']}\n";
echo "GC collected: {$stats['collected']}\n";
echo "GC threshold: {$stats['threshold']}\n";

// เปิด/ปิด GC
gc_disable(); // ปิด (ระวัง memory leak)
gc_enable();  // เปิด (default)

// PHP GC Configuration
ini_set('gc_probability', 1);    // ความน่าจะเป็นที่จะ Run GC
ini_set('gc_divisor', 100);      // GC จะ Run ทุก 1/100 requests = 1%
ini_set('gc_maxlifetime', 1440); // Session lifetime
```

### Memory Optimization Techniques

```php
<?php

// 1. ใช้ Generators แทน Arrays ขนาดใหญ่
function generateNumbers(int $start, int $end): \Generator
{
    for ($i = $start; $i <= $end; $i++) {
        yield $i;
    }
}

// Array: เก็บ 1M integers ~ 32MB
$array = range(1, 1_000_000); // 32MB+

// Generator: เกือบไม่ใช้ Memory
$gen = generateNumbers(1, 1_000_000); // ~0KB

foreach ($gen as $n) {
    // Process one at a time
}

// 2. Lazy Collections
use Illuminate\Support\LazyCollection;

LazyCollection::make(function () {
    $handle = fopen('huge-file.csv', 'r');
    while ($line = fgets($handle)) {
        yield str_getcsv($line);
    }
    fclose($handle);
})->filter(fn($row) => $row[2] > 0)
  ->map(fn($row) => processRow($row))
  ->each(fn($result) => saveResult($result));

// 3. SplFixedArray - เร็วและ Memory Efficient
$arr = new \SplFixedArray(1000);
for ($i = 0; $i < 1000; $i++) {
    $arr[$i] = $i * 2;
}

// SplFixedArray ใช้ Memory น้อยกว่า PHP Array ~30%
// เพราะไม่มี Hash Table Overhead

// 4. String Interning
// PHP Internals เก็บ String เดียวกันไว้ที่เดียว
$a = 'hello'; // stored in interned strings pool
$b = 'hello'; // same pointer as $a

// 5. Typed Properties ใช้ Memory น้อยกว่า
class OptimizedClass
{
    public int $count = 0;      // ใช้น้อยกว่า
    public float $total = 0.0;  // ใช้น้อยกว่า
    public ?string $name;       // Nullable เพิ่ม overhead เล็กน้อย
}

// เทียบกับ
class UnoptimizedClass
{
    public $count;  // Mixed type ใช้ zval เต็ม
    public $total;  
    public $name;   
}
```

---

## Fibers (PHP 8.1+)

Fibers เป็น Lightweight Coroutines ที่ช่วยให้เขียน Cooperative Multitasking

```php
<?php

// Fiber คือ "หยุดชั่วคราว" และ "ต่อทีหลัง" ได้
$fiber = new \Fiber(function (): void {
    echo "Fiber: Started\n";
    
    $value = \Fiber::suspend('first suspension');
    echo "Fiber: Resumed with '{$value}'\n";
    
    $value = \Fiber::suspend('second suspension');
    echo "Fiber: Resumed again with '{$value}'\n";
    
    echo "Fiber: Finished\n";
});

// Start Fiber
$suspended = $fiber->start();
echo "Main: Fiber suspended with '{$suspended}'\n";

// Resume Fiber
$suspended = $fiber->resume('hello');
echo "Main: Fiber suspended again with '{$suspended}'\n";

// Resume again
$fiber->resume('world');

echo "Main: Done\n";

/*
Output:
Fiber: Started
Main: Fiber suspended with 'first suspension'
Fiber: Resumed with 'hello'
Main: Fiber suspended again with 'second suspension'
Fiber: Resumed again with 'world'
Fiber: Finished
Main: Done
*/

// Practical: Async-like programming
class Scheduler
{
    private array $fibers = [];
    
    public function schedule(\Fiber $fiber): void
    {
        $this->fibers[] = $fiber;
    }
    
    public function run(): void
    {
        while (!empty($this->fibers)) {
            foreach ($this->fibers as $key => $fiber) {
                if ($fiber->isTerminated()) {
                    unset($this->fibers[$key]);
                    continue;
                }
                
                if ($fiber->isSuspended()) {
                    $fiber->resume();
                } elseif (!$fiber->isStarted()) {
                    $fiber->start();
                }
            }
        }
    }
}

$scheduler = new Scheduler();

$scheduler->schedule(new \Fiber(function () {
    for ($i = 1; $i <= 3; $i++) {
        echo "Task 1: Step {$i}\n";
        \Fiber::suspend();
    }
}));

$scheduler->schedule(new \Fiber(function () {
    for ($i = 1; $i <= 3; $i++) {
        echo "Task 2: Step {$i}\n";
        \Fiber::suspend();
    }
}));

$scheduler->run();

/*
Task 1: Step 1
Task 2: Step 1
Task 1: Step 2
Task 2: Step 2
Task 1: Step 3
Task 2: Step 3
*/
```

---

## PHP FFI (Foreign Function Interface)

```php
<?php

// PHP 7.4+ FFI ช่วยให้เรียก C Functions ได้โดยตรง
$ffi = FFI::cdef("
    // C Function declarations
    int abs(int x);
    double sqrt(double x);
    int printf(const char *fmt, ...);
    
    typedef struct {
        int x;
        int y;
    } Point;
", "libc.so.6");

echo $ffi->abs(-42) . "\n";   // 42
echo $ffi->sqrt(16.0) . "\n"; // 4

// สร้าง C Struct
$point = $ffi->new("Point");
$point->x = 10;
$point->y = 20;

echo "Point: ({$point->x}, {$point->y})\n";

// เรียก Custom Shared Library
$myLib = FFI::cdef("
    int add_numbers(int a, int b);
    char* reverse_string(const char *str);
", "./mylib.so");

echo $myLib->add_numbers(3, 4) . "\n"; // 7
```

---

## SPL Data Structures

```php
<?php

// SplStack - LIFO
$stack = new \SplStack();
$stack->push('first');
$stack->push('second');
$stack->push('third');

echo $stack->top() . "\n"; // 'third'
echo $stack->pop() . "\n"; // 'third'
echo $stack->pop() . "\n"; // 'second'

// SplQueue - FIFO
$queue = new \SplQueue();
$queue->enqueue('task1');
$queue->enqueue('task2');
$queue->enqueue('task3');

echo $queue->dequeue() . "\n"; // 'task1'

// SplHeap - Priority Queue
class MaxHeap extends \SplMaxHeap
{
    // เรียงจากมากไปน้อย
}

$heap = new MaxHeap();
$heap->insert(3);
$heap->insert(1);
$heap->insert(4);
$heap->insert(1);
$heap->insert(5);

while (!$heap->isEmpty()) {
    echo $heap->extract() . " "; // 5 4 3 1 1
}

// SplDoublyLinkedList
$dll = new \SplDoublyLinkedList();
$dll->push('a');
$dll->push('b');
$dll->push('c');

$dll->rewind();
while ($dll->valid()) {
    echo $dll->current() . " ";
    $dll->next();
}
// a b c

// SplObjectStorage - Object Map
$storage = new \SplObjectStorage();

$obj1 = new \stdClass();
$obj2 = new \stdClass();

$storage->attach($obj1, 'data for obj1');
$storage->attach($obj2, 'data for obj2');

echo $storage[$obj1] . "\n"; // "data for obj1"
echo count($storage) . "\n"; // 2
```

---

## สรุป PHP Internals

| ส่วน | หน้าที่ | ความสำคัญ |
|-----|--------|---------|
| Zend Engine | Core PHP Runtime | สูงมาก |
| OPcache | Cache Compiled Code | สูง (2-5x faster) |
| JIT | Machine Code Generation | สูง (CPU-bound) |
| Reference Counting | Memory Management | Automatic |
| Cycle Collector | Circular Reference GC | Automatic |
| Fibers | Cooperative Multitasking | PHP 8.1+ |
| FFI | C Interop | Advanced Use |

---

*การเข้าใจ PHP Internals ทำให้เขียน Code ที่ Efficient และแก้ Memory Issues ได้อย่างแม่นยำ*
