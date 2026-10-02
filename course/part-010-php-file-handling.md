# 📁 Part 10: PHP File Handling - การจัดการไฟล์

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- อ่านและเขียนไฟล์ด้วย PHP ได้
- ใช้ File System functions ได้อย่างคล่องแคล่ว
- จัดการ Directory operations ได้
- อ่านและเขียน CSV และ JSON ไฟล์ได้
- สร้าง File Manager อย่างง่ายได้

---

## 📌 1. การอ่านและเขียนไฟล์

### 1.1 การเขียนไฟล์

```php
<?php
// writing-files.php

// ======================================
// file_put_contents() - ง่ายและรวดเร็ว
// ======================================

// เขียนข้อความทับไฟล์เดิม
$result = file_put_contents('hello.txt', "สวัสดีโลก!\n");
echo "เขียน $result bytes\n";

// เพิ่มข้อความต่อท้ายไฟล์ (FILE_APPEND)
file_put_contents('hello.txt', "บรรทัดที่สอง\n", FILE_APPEND);
file_put_contents('hello.txt', "บรรทัดที่สาม\n", FILE_APPEND);

// เขียนหลายบรรทัดพร้อมกัน
$lines = ["line 1\n", "line 2\n", "line 3\n"];
file_put_contents('multiline.txt', $lines);

// ======================================
// fopen/fwrite/fclose - ควบคุมได้มากกว่า
// ======================================

// โหมดการเปิดไฟล์
// 'r'  - อ่านอย่างเดียว (ไฟล์ต้องมีอยู่)
// 'w'  - เขียน (สร้างใหม่หรือล้างไฟล์เดิม)
// 'a'  - เพิ่มต่อท้าย (สร้างถ้าไม่มี)
// 'x'  - สร้างใหม่ (ล้มเหลวถ้ามีอยู่แล้ว)
// 'r+' - อ่านและเขียน (ไฟล์ต้องมีอยู่)
// 'w+' - อ่านและเขียน (สร้างใหม่หรือล้าง)
// 'a+' - อ่านและเพิ่มต่อท้าย

$handle = fopen('log.txt', 'a');

if ($handle === false) {
    die("ไม่สามารถเปิดไฟล์ได้\n");
}

$timestamp = date('Y-m-d H:i:s');
fwrite($handle, "[$timestamp] บันทึกการทำงาน\n");
fwrite($handle, "[$timestamp] ผู้ใช้: admin เข้าสู่ระบบ\n");

fclose($handle);

echo "เขียนไฟล์สำเร็จ\n";

// ======================================
// การล็อคไฟล์ (File Locking)
// ======================================

// ป้องกัน race condition เมื่อหลาย process เขียนพร้อมกัน
$handle = fopen('counter.txt', 'a+');

if (flock($handle, LOCK_EX)) { // Exclusive lock
    $value = (int)fgets($handle);
    $value++;
    
    ftruncate($handle, 0);
    rewind($handle);
    fwrite($handle, $value);
    
    flock($handle, LOCK_UN); // Unlock
}

fclose($handle);
?>
```

### 1.2 การอ่านไฟล์

```php
<?php
// reading-files.php

// ======================================
// file_get_contents() - อ่านทั้งไฟล์
// ======================================

if (file_exists('hello.txt')) {
    $content = file_get_contents('hello.txt');
    echo $content;
}

// อ่านจาก URL (ถ้า allow_url_fopen = On)
// $html = file_get_contents('https://example.com');

// ======================================
// file() - อ่านเป็น array ของบรรทัด
// ======================================

$lines = file('hello.txt', FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES);

foreach ($lines as $lineNumber => $line) {
    echo "บรรทัด " . ($lineNumber + 1) . ": $line\n";
}

// ======================================
// fopen/fgets - อ่านทีละบรรทัด (ประหยัด memory)
// ======================================

$handle = fopen('large-file.txt', 'r');

if ($handle) {
    $lineCount = 0;
    
    while (($line = fgets($handle, 4096)) !== false) {
        $lineCount++;
        $line = trim($line);
        
        // ประมวลผลทีละบรรทัด
        if (!empty($line)) {
            echo "Line $lineCount: $line\n";
        }
        
        // หยุดถ้าอ่านครบ 100 บรรทัด
        if ($lineCount >= 100) break;
    }
    
    fclose($handle);
}

// ======================================
// readfile() - อ่านและ output ไปยัง browser
// ======================================

// ดาวน์โหลดไฟล์
header('Content-Type: application/octet-stream');
header('Content-Disposition: attachment; filename="download.txt"');
readfile('hello.txt');

// ======================================
// SplFileObject - OOP approach
// ======================================

$file = new SplFileObject('hello.txt', 'r');

foreach ($file as $lineNumber => $line) {
    echo "Line " . ($lineNumber + 1) . ": " . trim($line) . "\n";
}
?>
```

### 1.3 Logger Class

```php
<?php
// Logger.php

class Logger
{
    private string $logFile;
    private string $logLevel;
    private int $maxFileSize;
    
    const LEVELS = ['DEBUG', 'INFO', 'WARNING', 'ERROR', 'CRITICAL'];
    
    public function __construct(
        string $logFile = 'app.log',
        string $logLevel = 'DEBUG',
        int $maxFileSize = 5 * 1024 * 1024 // 5MB
    ) {
        $this->logFile = $logFile;
        $this->logLevel = $logLevel;
        $this->maxFileSize = $maxFileSize;
        
        // สร้าง directory ถ้าไม่มี
        $dir = dirname($logFile);
        if (!is_dir($dir)) {
            mkdir($dir, 0755, true);
        }
    }
    
    public function debug(string $message, array $context = []): void
    {
        $this->log('DEBUG', $message, $context);
    }
    
    public function info(string $message, array $context = []): void
    {
        $this->log('INFO', $message, $context);
    }
    
    public function warning(string $message, array $context = []): void
    {
        $this->log('WARNING', $message, $context);
    }
    
    public function error(string $message, array $context = []): void
    {
        $this->log('ERROR', $message, $context);
    }
    
    public function critical(string $message, array $context = []): void
    {
        $this->log('CRITICAL', $message, $context);
    }
    
    private function log(string $level, string $message, array $context = []): void
    {
        // ตรวจสอบ level
        if (array_search($level, self::LEVELS) < array_search($this->logLevel, self::LEVELS)) {
            return;
        }
        
        // Rotate log ถ้าไฟล์ใหญ่เกินไป
        $this->rotateIfNeeded();
        
        // แทนที่ placeholders ใน message
        $message = $this->interpolate($message, $context);
        
        // สร้าง log entry
        $entry = sprintf(
            "[%s] [%s] %s%s\n",
            date('Y-m-d H:i:s'),
            $level,
            $message,
            !empty($context) ? ' ' . json_encode($context) : ''
        );
        
        file_put_contents($this->logFile, $entry, FILE_APPEND | LOCK_EX);
    }
    
    private function interpolate(string $message, array $context): string
    {
        $replace = [];
        foreach ($context as $key => $value) {
            $replace['{' . $key . '}'] = $value;
        }
        return strtr($message, $replace);
    }
    
    private function rotateIfNeeded(): void
    {
        if (file_exists($this->logFile) && filesize($this->logFile) > $this->maxFileSize) {
            $backupFile = $this->logFile . '.' . date('Y-m-d-H-i-s');
            rename($this->logFile, $backupFile);
        }
    }
    
    public function readLog(int $lines = 100): array
    {
        if (!file_exists($this->logFile)) {
            return [];
        }
        
        $allLines = file($this->logFile, FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES);
        return array_slice($allLines, -$lines);
    }
    
    public function clearLog(): void
    {
        file_put_contents($this->logFile, '');
    }
}

// ตัวอย่างการใช้งาน
$logger = new Logger('logs/app.log', 'INFO');

$logger->info('Application started');
$logger->info('User {username} logged in', ['username' => 'admin', 'ip' => '127.0.0.1']);
$logger->warning('Rate limit approaching for user {id}', ['id' => 42]);
$logger->error('Database connection failed', ['host' => 'localhost', 'port' => 3306]);

// อ่าน log ล่าสุด
$recentLogs = $logger->readLog(10);
foreach ($recentLogs as $log) {
    echo $log . "\n";
}
?>
```

---

## 📌 2. File System Functions

### 2.1 ฟังก์ชันพื้นฐาน

```php
<?php
// filesystem-functions.php

$file = 'test.txt';

// ======================================
// ตรวจสอบไฟล์
// ======================================
echo "file_exists: " . (file_exists($file) ? 'มี' : 'ไม่มี') . "\n";
echo "is_file: " . (is_file($file) ? 'ใช่' : 'ไม่ใช่') . "\n";
echo "is_dir: " . (is_dir($file) ? 'ใช่' : 'ไม่ใช่') . "\n";
echo "is_readable: " . (is_readable($file) ? 'ได้' : 'ไม่ได้') . "\n";
echo "is_writable: " . (is_writable($file) ? 'ได้' : 'ไม่ได้') . "\n";

// ======================================
// ข้อมูลไฟล์
// ======================================
if (file_exists($file)) {
    echo "ขนาด: " . filesize($file) . " bytes\n";
    echo "แก้ไขล่าสุด: " . date('d/m/Y H:i:s', filemtime($file)) . "\n";
    echo "เข้าถึงล่าสุด: " . date('d/m/Y H:i:s', fileatime($file)) . "\n";
    echo "สร้างเมื่อ: " . date('d/m/Y H:i:s', filectime($file)) . "\n";
    
    $info = pathinfo($file);
    echo "Directory: " . $info['dirname'] . "\n";
    echo "Filename: " . $info['filename'] . "\n";
    echo "Extension: " . $info['extension'] . "\n";
    echo "Basename: " . $info['basename'] . "\n";
}

// ======================================
// คัดลอก ย้าย ลบ
// ======================================

// คัดลอกไฟล์
if (copy('source.txt', 'destination.txt')) {
    echo "คัดลอกสำเร็จ\n";
}

// เปลี่ยนชื่อหรือย้ายไฟล์
if (rename('old-name.txt', 'new-name.txt')) {
    echo "เปลี่ยนชื่อสำเร็จ\n";
}

// ลบไฟล์
if (file_exists('temp.txt') && unlink('temp.txt')) {
    echo "ลบไฟล์สำเร็จ\n";
}

// ======================================
// การค้นหาไฟล์ด้วย glob()
// ======================================

// หาไฟล์ .php ทั้งหมด
$phpFiles = glob('*.php');
echo "PHP files: " . count($phpFiles) . "\n";
foreach ($phpFiles as $phpFile) {
    echo "  - $phpFile\n";
}

// หาไฟล์หลายประเภท
$files = array_merge(
    glob('*.php') ?: [],
    glob('*.html') ?: [],
    glob('*.txt') ?: []
);

// หาแบบ recursive
function globRecursive(string $pattern, int $flags = 0): array
{
    $files = glob($pattern, $flags) ?: [];
    
    foreach (glob(dirname($pattern) . '/*', GLOB_ONLYDIR | GLOB_NOSORT) as $dir) {
        $files = array_merge($files, globRecursive($dir . '/' . basename($pattern), $flags));
    }
    
    return $files;
}

$allPhpFiles = globRecursive('**/*.php');

// ======================================
// File Info
// ======================================

function formatBytes(int $bytes, int $precision = 2): string
{
    $units = ['B', 'KB', 'MB', 'GB', 'TB'];
    $i = 0;
    
    while ($bytes >= 1024 && $i < count($units) - 1) {
        $bytes /= 1024;
        $i++;
    }
    
    return round($bytes, $precision) . ' ' . $units[$i];
}

function getFileInfo(string $path): array
{
    if (!file_exists($path)) {
        return [];
    }
    
    $stat = stat($path);
    $pathInfo = pathinfo($path);
    
    return [
        'name'         => $pathInfo['basename'],
        'extension'    => $pathInfo['extension'] ?? '',
        'directory'    => $pathInfo['dirname'],
        'size'         => $stat['size'],
        'size_human'   => formatBytes($stat['size']),
        'modified'     => date('Y-m-d H:i:s', $stat['mtime']),
        'created'      => date('Y-m-d H:i:s', $stat['ctime']),
        'is_readable'  => is_readable($path),
        'is_writable'  => is_writable($path),
        'permissions'  => substr(sprintf('%o', fileperms($path)), -4),
    ];
}

$info = getFileInfo('hello.txt');
print_r($info);
?>
```

---

## 📌 3. Directory Operations

### 3.1 การจัดการ Directory

```php
<?php
// directory-operations.php

// ======================================
// สร้าง Directory
// ======================================

// สร้าง directory เดี่ยว
if (!is_dir('mydir')) {
    mkdir('mydir', 0755);
}

// สร้าง directory แบบ recursive
if (!is_dir('parent/child/grandchild')) {
    mkdir('parent/child/grandchild', 0755, true);
}

// ======================================
// ลิสต์ไฟล์ใน Directory
// ======================================

// วิธีที่ 1: scandir()
$items = scandir('.');
foreach ($items as $item) {
    if ($item !== '.' && $item !== '..') {
        $type = is_dir($item) ? 'dir' : 'file';
        echo "[$type] $item\n";
    }
}

// วิธีที่ 2: DirectoryIterator (OOP)
$dir = new DirectoryIterator('.');

foreach ($dir as $fileInfo) {
    if ($fileInfo->isDot()) continue;
    
    echo sprintf(
        "[%s] %-30s %s\n",
        $fileInfo->isDir() ? 'DIR ' : 'FILE',
        $fileInfo->getFilename(),
        $fileInfo->isFile() ? formatBytes($fileInfo->getSize()) : ''
    );
}

// วิธีที่ 3: RecursiveDirectoryIterator
function listDirectoryRecursive(string $path, int $depth = 0): void
{
    $iterator = new RecursiveDirectoryIterator($path, RecursiveDirectoryIterator::SKIP_DOTS);
    $recursive = new RecursiveIteratorIterator($iterator, RecursiveIteratorIterator::SELF_FIRST);
    
    foreach ($recursive as $file) {
        $indent = str_repeat('  ', $recursive->getDepth());
        $icon = $file->isDir() ? '📁' : '📄';
        echo $indent . $icon . ' ' . $file->getFilename() . "\n";
    }
}

listDirectoryRecursive('.');

// ======================================
// คัดลอก Directory
// ======================================

function copyDirectory(string $source, string $destination): bool
{
    if (!is_dir($source)) return false;
    
    if (!is_dir($destination)) {
        mkdir($destination, 0755, true);
    }
    
    $iterator = new RecursiveDirectoryIterator($source, RecursiveDirectoryIterator::SKIP_DOTS);
    $recursive = new RecursiveIteratorIterator($iterator, RecursiveIteratorIterator::SELF_FIRST);
    
    foreach ($recursive as $item) {
        $targetPath = $destination . DIRECTORY_SEPARATOR . $recursive->getSubPathName();
        
        if ($item->isDir()) {
            if (!is_dir($targetPath)) {
                mkdir($targetPath, 0755, true);
            }
        } else {
            copy($item->getPathname(), $targetPath);
        }
    }
    
    return true;
}

// ======================================
// ลบ Directory แบบ Recursive
// ======================================

function deleteDirectory(string $path): bool
{
    if (!is_dir($path)) return false;
    
    $items = new RecursiveIteratorIterator(
        new RecursiveDirectoryIterator($path, RecursiveDirectoryIterator::SKIP_DOTS),
        RecursiveIteratorIterator::CHILD_FIRST
    );
    
    foreach ($items as $item) {
        if ($item->isDir()) {
            rmdir($item->getPathname());
        } else {
            unlink($item->getPathname());
        }
    }
    
    return rmdir($path);
}

// ======================================
// คำนวณขนาด Directory
// ======================================

function getDirectorySize(string $path): int
{
    $size = 0;
    
    $iterator = new RecursiveIteratorIterator(
        new RecursiveDirectoryIterator($path, RecursiveDirectoryIterator::SKIP_DOTS)
    );
    
    foreach ($iterator as $file) {
        if ($file->isFile()) {
            $size += $file->getSize();
        }
    }
    
    return $size;
}

$size = getDirectorySize('.');
echo "ขนาด directory: " . formatBytes($size) . "\n";
?>
```

---

## 📌 4. CSV/JSON File Handling

### 4.1 การทำงานกับ CSV

```php
<?php
// csv-handler.php

class CsvHandler
{
    private string $delimiter;
    private string $enclosure;
    private string $escape;
    
    public function __construct(
        string $delimiter = ',',
        string $enclosure = '"',
        string $escape = '\\'
    ) {
        $this->delimiter = $delimiter;
        $this->enclosure = $enclosure;
        $this->escape = $escape;
    }
    
    /**
     * อ่าน CSV file
     */
    public function read(string $filepath, bool $hasHeader = true): array
    {
        if (!file_exists($filepath)) {
            throw new RuntimeException("File not found: $filepath");
        }
        
        $rows = [];
        $headers = [];
        
        $handle = fopen($filepath, 'r');
        
        if ($handle === false) {
            throw new RuntimeException("Cannot open file: $filepath");
        }
        
        // ตั้งค่า encoding สำหรับภาษาไทย
        stream_filter_prepend($handle, 'convert.iconv.UTF-8/UTF-8//IGNORE');
        
        $lineNumber = 0;
        
        while (($row = fgetcsv($handle, 0, $this->delimiter, $this->enclosure, $this->escape)) !== false) {
            $lineNumber++;
            
            if ($hasHeader && $lineNumber === 1) {
                $headers = $row;
                continue;
            }
            
            if ($hasHeader && !empty($headers)) {
                // สร้าง associative array
                $rows[] = array_combine($headers, $row);
            } else {
                $rows[] = $row;
            }
        }
        
        fclose($handle);
        
        return $rows;
    }
    
    /**
     * เขียน CSV file
     */
    public function write(string $filepath, array $data, array $headers = []): bool
    {
        $handle = fopen($filepath, 'w');
        
        if ($handle === false) {
            throw new RuntimeException("Cannot create file: $filepath");
        }
        
        // เพิ่ม BOM สำหรับ Excel (UTF-8)
        fputs($handle, "\xEF\xBB\xBF");
        
        // เขียน headers
        if (!empty($headers)) {
            fputcsv($handle, $headers, $this->delimiter, $this->enclosure);
        } elseif (!empty($data)) {
            // ใช้ keys ของแถวแรกเป็น headers
            fputcsv($handle, array_keys($data[0]), $this->delimiter, $this->enclosure);
        }
        
        // เขียนข้อมูล
        foreach ($data as $row) {
            fputcsv($handle, $row, $this->delimiter, $this->enclosure);
        }
        
        fclose($handle);
        
        return true;
    }
    
    /**
     * Append ข้อมูลต่อท้าย CSV
     */
    public function append(string $filepath, array $row): bool
    {
        $handle = fopen($filepath, 'a');
        
        if ($handle === false) {
            return false;
        }
        
        fputcsv($handle, $row, $this->delimiter, $this->enclosure);
        fclose($handle);
        
        return true;
    }
    
    /**
     * ส่งออก CSV ให้ browser download
     */
    public function download(array $data, string $filename = 'export.csv', array $headers = []): void
    {
        header('Content-Type: text/csv; charset=utf-8');
        header('Content-Disposition: attachment; filename="' . $filename . '"');
        header('Pragma: no-cache');
        header('Expires: 0');
        
        $output = fopen('php://output', 'w');
        
        // BOM สำหรับ Excel
        fputs($output, "\xEF\xBB\xBF");
        
        if (!empty($headers)) {
            fputcsv($output, $headers);
        } elseif (!empty($data)) {
            fputcsv($output, array_keys($data[0]));
        }
        
        foreach ($data as $row) {
            fputcsv($output, $row);
        }
        
        fclose($output);
        exit;
    }
}

// ======================================
// ตัวอย่างการใช้งาน
// ======================================

$csv = new CsvHandler();

// ข้อมูลตัวอย่าง
$students = [
    ['id' => 1, 'name' => 'สมชาย ใจดี', 'email' => 'somchai@email.com', 'score' => 85],
    ['id' => 2, 'name' => 'สมหญิง รักเรียน', 'email' => 'somying@email.com', 'score' => 92],
    ['id' => 3, 'name' => 'วิชัย เก่งมาก', 'email' => 'wichai@email.com', 'score' => 78],
    ['id' => 4, 'name' => 'มานี มีความสุข', 'email' => 'manee@email.com', 'score' => 95],
];

// เขียน CSV
$csv->write('students.csv', $students, ['รหัส', 'ชื่อ', 'อีเมล', 'คะแนน']);

// อ่าน CSV
$data = $csv->read('students.csv');
foreach ($data as $row) {
    echo $row['ชื่อ'] . ": " . $row['คะแนน'] . " คะแนน\n";
}

// Download CSV
// $csv->download($students, 'students.csv', ['รหัส', 'ชื่อ', 'อีเมล', 'คะแนน']);
?>
```

### 4.2 การทำงานกับ JSON

```php
<?php
// json-handler.php

class JsonStorage
{
    private string $filepath;
    
    public function __construct(string $filepath)
    {
        $this->filepath = $filepath;
        
        $dir = dirname($filepath);
        if (!is_dir($dir)) {
            mkdir($dir, 0755, true);
        }
        
        if (!file_exists($filepath)) {
            file_put_contents($filepath, '[]');
        }
    }
    
    /**
     * อ่านข้อมูลทั้งหมด
     */
    public function all(): array
    {
        $content = file_get_contents($this->filepath);
        $data = json_decode($content, true);
        
        if (json_last_error() !== JSON_ERROR_NONE) {
            throw new RuntimeException('Invalid JSON: ' . json_last_error_msg());
        }
        
        return $data ?? [];
    }
    
    /**
     * หาข้อมูลด้วย ID
     */
    public function find(int|string $id): ?array
    {
        $data = $this->all();
        
        foreach ($data as $item) {
            if (isset($item['id']) && $item['id'] == $id) {
                return $item;
            }
        }
        
        return null;
    }
    
    /**
     * ค้นหาข้อมูล
     */
    public function where(string $field, mixed $value): array
    {
        return array_filter($this->all(), fn($item) => isset($item[$field]) && $item[$field] == $value);
    }
    
    /**
     * เพิ่มข้อมูล
     */
    public function insert(array $record): array
    {
        $data = $this->all();
        
        // สร้าง ID อัตโนมัติ
        if (!isset($record['id'])) {
            $maxId = empty($data) ? 0 : max(array_column($data, 'id'));
            $record['id'] = $maxId + 1;
        }
        
        $record['created_at'] = date('Y-m-d H:i:s');
        $record['updated_at'] = date('Y-m-d H:i:s');
        
        $data[] = $record;
        $this->save($data);
        
        return $record;
    }
    
    /**
     * อัปเดตข้อมูล
     */
    public function update(int|string $id, array $updates): bool
    {
        $data = $this->all();
        $found = false;
        
        foreach ($data as &$item) {
            if (isset($item['id']) && $item['id'] == $id) {
                $item = array_merge($item, $updates);
                $item['updated_at'] = date('Y-m-d H:i:s');
                $found = true;
                break;
            }
        }
        
        if ($found) {
            $this->save($data);
        }
        
        return $found;
    }
    
    /**
     * ลบข้อมูล
     */
    public function delete(int|string $id): bool
    {
        $data = $this->all();
        $originalCount = count($data);
        
        $data = array_filter($data, fn($item) => !isset($item['id']) || $item['id'] != $id);
        
        if (count($data) < $originalCount) {
            $this->save(array_values($data));
            return true;
        }
        
        return false;
    }
    
    /**
     * บันทึกข้อมูลลงไฟล์
     */
    private function save(array $data): void
    {
        $json = json_encode($data, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES);
        
        if ($json === false) {
            throw new RuntimeException('Cannot encode JSON: ' . json_last_error_msg());
        }
        
        file_put_contents($this->filepath, $json, LOCK_EX);
    }
    
    /**
     * นับจำนวนรายการ
     */
    public function count(): int
    {
        return count($this->all());
    }
}

// ======================================
// ตัวอย่างการใช้งาน
// ======================================

$db = new JsonStorage('data/products.json');

// เพิ่มสินค้า
$product1 = $db->insert([
    'name'     => 'MacBook Pro',
    'price'    => 89000,
    'category' => 'notebook',
    'stock'    => 10,
]);

$product2 = $db->insert([
    'name'     => 'iPhone 15',
    'price'    => 45000,
    'category' => 'smartphone',
    'stock'    => 25,
]);

$product3 = $db->insert([
    'name'     => 'iPad Air',
    'price'    => 32000,
    'category' => 'tablet',
    'stock'    => 15,
]);

echo "เพิ่มสินค้า 3 รายการ\n";

// อ่านข้อมูลทั้งหมด
$products = $db->all();
echo "สินค้าทั้งหมด: " . count($products) . " รายการ\n";

// หาสินค้าด้วย ID
$found = $db->find(1);
echo "สินค้า ID 1: " . $found['name'] . " ราคา " . number_format($found['price']) . " บาท\n";

// ค้นหาตาม category
$notebooks = $db->where('category', 'notebook');
echo "Notebook: " . count($notebooks) . " รายการ\n";

// อัปเดต
$db->update(1, ['price' => 92000, 'stock' => 8]);
echo "อัปเดตราคา MacBook Pro เป็น 92,000 บาท\n";

// ลบ
$db->delete(3);
echo "ลบ iPad Air แล้ว\n";
echo "สินค้าที่เหลือ: " . $db->count() . " รายการ\n";
?>
```

---

## 🛠️ Workshop: สร้าง File Manager

```php
<?php
// file-manager.php

class FileManager
{
    private string $baseDir;
    private array $allowedExtensions;
    
    public function __construct(string $baseDir = 'files/')
    {
        $this->baseDir = rtrim(realpath($baseDir) ?: $baseDir, '/') . '/';
        $this->allowedExtensions = ['txt', 'pdf', 'jpg', 'jpeg', 'png', 'gif', 'csv', 'json', 'zip'];
        
        if (!is_dir($this->baseDir)) {
            mkdir($this->baseDir, 0755, true);
        }
    }
    
    /**
     * ลิสต์ไฟล์และโฟลเดอร์
     */
    public function listContents(string $subdir = ''): array
    {
        $path = $this->resolvePath($subdir);
        
        if (!is_dir($path)) {
            throw new RuntimeException("Directory not found: $subdir");
        }
        
        $items = [];
        $scan = scandir($path);
        
        foreach ($scan as $item) {
            if ($item === '.' || $item === '..') continue;
            
            $fullPath = $path . $item;
            $isDir = is_dir($fullPath);
            
            $items[] = [
                'name'      => $item,
                'type'      => $isDir ? 'directory' : 'file',
                'size'      => $isDir ? null : filesize($fullPath),
                'size_human'=> $isDir ? null : $this->formatBytes(filesize($fullPath)),
                'modified'  => date('Y-m-d H:i:s', filemtime($fullPath)),
                'extension' => $isDir ? null : pathinfo($item, PATHINFO_EXTENSION),
                'path'      => ltrim($subdir . '/' . $item, '/'),
            ];
        }
        
        // เรียงโฟลเดอร์ก่อนไฟล์
        usort($items, function($a, $b) {
            if ($a['type'] !== $b['type']) {
                return $a['type'] === 'directory' ? -1 : 1;
            }
            return strcasecmp($a['name'], $b['name']);
        });
        
        return $items;
    }
    
    /**
     * อ่านเนื้อหาไฟล์
     */
    public function readFile(string $filepath): string
    {
        $path = $this->resolvePath($filepath);
        
        if (!file_exists($path) || !is_file($path)) {
            throw new RuntimeException("File not found: $filepath");
        }
        
        return file_get_contents($path);
    }
    
    /**
     * เขียนไฟล์
     */
    public function writeFile(string $filepath, string $content): bool
    {
        $path = $this->resolvePath($filepath);
        
        $ext = strtolower(pathinfo($path, PATHINFO_EXTENSION));
        if (!in_array($ext, $this->allowedExtensions)) {
            throw new RuntimeException("Extension not allowed: $ext");
        }
        
        $dir = dirname($path);
        if (!is_dir($dir)) {
            mkdir($dir, 0755, true);
        }
        
        return file_put_contents($path, $content, LOCK_EX) !== false;
    }
    
    /**
     * ลบไฟล์
     */
    public function deleteFile(string $filepath): bool
    {
        $path = $this->resolvePath($filepath);
        
        if (!file_exists($path)) {
            throw new RuntimeException("File not found: $filepath");
        }
        
        if (is_dir($path)) {
            return $this->deleteDirectory($path);
        }
        
        return unlink($path);
    }
    
    /**
     * สร้างโฟลเดอร์
     */
    public function createDirectory(string $dirpath): bool
    {
        $path = $this->resolvePath($dirpath);
        
        if (is_dir($path)) {
            return true;
        }
        
        return mkdir($path, 0755, true);
    }
    
    /**
     * เปลี่ยนชื่อไฟล์/โฟลเดอร์
     */
    public function rename(string $oldPath, string $newName): bool
    {
        $oldFullPath = $this->resolvePath($oldPath);
        $newFullPath = dirname($oldFullPath) . '/' . $newName;
        
        if (!file_exists($oldFullPath)) {
            throw new RuntimeException("File not found: $oldPath");
        }
        
        return rename($oldFullPath, $newFullPath);
    }
    
    /**
     * ค้นหาไฟล์
     */
    public function search(string $query, string $subdir = ''): array
    {
        $path = $this->resolvePath($subdir);
        $results = [];
        
        $iterator = new RecursiveIteratorIterator(
            new RecursiveDirectoryIterator($path, RecursiveDirectoryIterator::SKIP_DOTS)
        );
        
        foreach ($iterator as $file) {
            if (stripos($file->getFilename(), $query) !== false) {
                $results[] = [
                    'name'     => $file->getFilename(),
                    'path'     => $file->getPathname(),
                    'relative' => str_replace($this->baseDir, '', $file->getPathname()),
                    'type'     => $file->isDir() ? 'directory' : 'file',
                    'size'     => $file->isFile() ? $file->getSize() : null,
                ];
            }
        }
        
        return $results;
    }
    
    /**
     * ดาวน์โหลดไฟล์
     */
    public function download(string $filepath): void
    {
        $path = $this->resolvePath($filepath);
        
        if (!file_exists($path) || !is_file($path)) {
            throw new RuntimeException("File not found: $filepath");
        }
        
        $filename = basename($path);
        $size = filesize($path);
        
        header('Content-Type: application/octet-stream');
        header('Content-Disposition: attachment; filename="' . $filename . '"');
        header('Content-Length: ' . $size);
        header('Pragma: no-cache');
        
        readfile($path);
        exit;
    }
    
    /**
     * ตรวจสอบ path ไม่ให้ออกนอก base dir (Path Traversal Prevention)
     */
    private function resolvePath(string $path): string
    {
        // ลบ path traversal attempts
        $path = str_replace(['../', '..\\', '..'], '', $path);
        $fullPath = $this->baseDir . ltrim($path, '/');
        
        // ตรวจสอบว่า path อยู่ใน baseDir
        $realPath = realpath($fullPath) ?: $fullPath;
        
        if (strpos($realPath . '/', $this->baseDir) !== 0) {
            throw new RuntimeException("Access denied: path outside base directory");
        }
        
        return $fullPath;
    }
    
    private function deleteDirectory(string $path): bool
    {
        $items = new RecursiveIteratorIterator(
            new RecursiveDirectoryIterator($path, RecursiveDirectoryIterator::SKIP_DOTS),
            RecursiveIteratorIterator::CHILD_FIRST
        );
        
        foreach ($items as $item) {
            $item->isDir() ? rmdir($item->getPathname()) : unlink($item->getPathname());
        }
        
        return rmdir($path);
    }
    
    private function formatBytes(int $bytes): string
    {
        $units = ['B', 'KB', 'MB', 'GB'];
        $i = 0;
        while ($bytes >= 1024 && $i < count($units) - 1) {
            $bytes /= 1024;
            $i++;
        }
        return round($bytes, 2) . ' ' . $units[$i];
    }
}

// ======================================
// Web Interface
// ======================================

$fm = new FileManager('files/');
$action = $_GET['action'] ?? 'list';
$currentPath = $_GET['path'] ?? '';

try {
    switch ($action) {
        case 'list':
            $items = $fm->listContents($currentPath);
            break;
            
        case 'delete':
            $file = $_GET['file'] ?? '';
            $fm->deleteFile($file);
            header('Location: ?action=list&path=' . $currentPath);
            exit;
            
        case 'mkdir':
            $dirname = $_POST['dirname'] ?? '';
            $fm->createDirectory($currentPath . '/' . $dirname);
            header('Location: ?action=list&path=' . $currentPath);
            exit;
            
        case 'download':
            $file = $_GET['file'] ?? '';
            $fm->download($file);
            break;
            
        case 'search':
            $query = $_GET['q'] ?? '';
            $items = $fm->search($query);
            break;
    }
} catch (Exception $e) {
    $error = $e->getMessage();
}
?>

<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>File Manager</title>
    <style>
        body { font-family: monospace; padding: 20px; }
        table { width: 100%; border-collapse: collapse; }
        th, td { padding: 8px; text-align: left; border-bottom: 1px solid #ddd; }
        th { background: #f5f5f5; }
        .dir { color: #0066cc; font-weight: bold; }
        .actions a { margin-right: 10px; color: #666; }
        .actions a:hover { color: #333; }
        .path-nav { background: #f9f9f9; padding: 10px; margin-bottom: 20px; border-radius: 4px; }
    </style>
</head>
<body>
    <h1>📁 File Manager</h1>
    
    <?php if (isset($error)): ?>
        <div style="color: red; padding: 10px; background: #ffeeee; margin-bottom: 20px;">
            ❌ <?= htmlspecialchars($error) ?>
        </div>
    <?php endif; ?>
    
    <!-- Breadcrumb -->
    <div class="path-nav">
        📍 <a href="?action=list">Home</a>
        <?php
        if ($currentPath) {
            $parts = explode('/', $currentPath);
            $accumulated = '';
            foreach ($parts as $part) {
                if ($part) {
                    $accumulated .= '/' . $part;
                    echo " / <a href='?action=list&path=" . ltrim($accumulated, '/') . "'>" . htmlspecialchars($part) . "</a>";
                }
            }
        }
        ?>
    </div>
    
    <!-- ค้นหา -->
    <form action="" method="get" style="margin-bottom: 20px;">
        <input type="hidden" name="action" value="search">
        <input type="text" name="q" placeholder="ค้นหาไฟล์..." value="<?= htmlspecialchars($_GET['q'] ?? '') ?>">
        <button type="submit">🔍 ค้นหา</button>
    </form>
    
    <!-- สร้างโฟลเดอร์ใหม่ -->
    <form action="" method="post" style="margin-bottom: 20px;">
        <input type="hidden" name="action" value="mkdir">
        <input type="hidden" name="path" value="<?= htmlspecialchars($currentPath) ?>">
        <input type="text" name="dirname" placeholder="ชื่อโฟลเดอร์ใหม่">
        <button type="submit">📁 สร้างโฟลเดอร์</button>
    </form>
    
    <!-- รายการไฟล์ -->
    <?php if (isset($items)): ?>
    <table>
        <thead>
            <tr>
                <th>ชื่อ</th>
                <th>ประเภท</th>
                <th>ขนาด</th>
                <th>แก้ไขล่าสุด</th>
                <th>การดำเนินการ</th>
            </tr>
        </thead>
        <tbody>
            <?php foreach ($items as $item): ?>
            <tr>
                <td>
                    <?php if ($item['type'] === 'directory'): ?>
                        📁 <a href="?action=list&path=<?= urlencode($item['path']) ?>" class="dir">
                            <?= htmlspecialchars($item['name']) ?>
                        </a>
                    <?php else: ?>
                        📄 <?= htmlspecialchars($item['name']) ?>
                    <?php endif; ?>
                </td>
                <td><?= $item['type'] === 'directory' ? 'โฟลเดอร์' : strtoupper($item['extension'] ?? '') ?></td>
                <td><?= $item['size_human'] ?? '-' ?></td>
                <td><?= $item['modified'] ?></td>
                <td class="actions">
                    <?php if ($item['type'] === 'file'): ?>
                        <a href="?action=download&file=<?= urlencode($item['path']) ?>">⬇️ ดาวน์โหลด</a>
                    <?php endif; ?>
                    <a href="?action=delete&file=<?= urlencode($item['path']) ?>&path=<?= urlencode($currentPath) ?>"
                       onclick="return confirm('ยืนยันการลบ?')">🗑️ ลบ</a>
                </td>
            </tr>
            <?php endforeach; ?>
            
            <?php if (empty($items)): ?>
            <tr>
                <td colspan="5" style="text-align: center; color: #666;">ไม่มีไฟล์ในโฟลเดอร์นี้</td>
            </tr>
            <?php endif; ?>
        </tbody>
    </table>
    <?php endif; ?>
</body>
</html>
```

---

## 📝 Quiz

### คำถาม

**ข้อ 1:** Flag ใดใน `file_put_contents()` ใช้สำหรับเพิ่มข้อความต่อท้ายไฟล์?
- A. `FILE_APPEND`
- B. `FILE_ADD`
- C. `APPEND_FILE`
- D. `FILE_END`

**ข้อ 2:** ฟังก์ชันใดใช้อ่านไฟล์ทั้งหมดเป็น array ของบรรทัด?
- A. `file_get_contents()`
- B. `fread()`
- C. `file()`
- D. `fgets()`

**ข้อ 3:** ฟังก์ชันใดใช้สร้างโฟลเดอร์แบบ recursive?
- A. `mkdir('a/b/c', 0755)`
- B. `mkdir('a/b/c', 0755, true)`
- C. `makedir('a/b/c')`
- D. `createdir('a/b/c', true)`

**ข้อ 4:** Path Traversal Attack คืออะไร?
- A. การเข้าถึงไฟล์นอก directory ที่อนุญาต
- B. การเขียน path ที่ยาวเกินไป
- C. การคัดลอกไฟล์ไปยัง directory อื่น
- D. การลบ directory แบบ recursive

**ข้อ 5:** BOM (Byte Order Mark) `\xEF\xBB\xBF` ใน CSV ใช้ทำอะไร?
- A. ระบุขนาดไฟล์
- B. บอกให้ Excel รู้ว่าเป็น UTF-8
- C. ป้องกันการแก้ไขไฟล์
- D. เพิ่มความเร็วในการอ่าน

### เฉลย

| ข้อ | คำตอบ | อธิบาย |
|-----|-------|--------|
| 1 | A | `FILE_APPEND` flag ทำให้เพิ่มข้อมูลต่อท้ายแทนที่จะเขียนทับ |
| 2 | C | `file()` อ่านไฟล์และ return เป็น array โดยแต่ละ element คือหนึ่งบรรทัด |
| 3 | B | parameter ที่ 3 ของ `mkdir()` เป็น `true` ทำให้สร้าง parent directories ด้วย |
| 4 | A | Path Traversal ใช้ `../` เพื่อออกจาก directory ที่กำหนดและเข้าถึงไฟล์อื่น |
| 5 | B | BOM บอก Excel ว่าไฟล์ CSV เป็น UTF-8 ทำให้แสดงภาษาไทยได้ถูกต้อง |

---

## ➡️ Part ถัดไป

**[Part 11: PHP Sessions & Cookies - การจัดการ Session และ Cookie](part-011-php-sessions-cookies.md)**
