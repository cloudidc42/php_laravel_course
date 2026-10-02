# 🔌 Part 13: PHP PDO - PHP Data Objects

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เชื่อมต่อ Database ด้วย PDO ได้
- ทำ CRUD operations ผ่าน PDO ได้
- ใช้ PDO Transactions ได้
- สร้าง Query Builder แบบง่ายได้
- สร้าง Data Access Layer ที่ดีได้

---

## 📌 1. PDO Connection

### 1.1 PDO คืออะไร และทำไมถึงดีกว่า MySQLi?

| คุณสมบัติ | PDO | MySQLi |
|-----------|-----|--------|
| รองรับ Database | 12+ databases | MySQL เท่านั้น |
| API | OOP เท่านั้น | OOP + Procedural |
| Named Parameters | ✅ | ❌ |
| การเปลี่ยน Database | ง่าย | ต้องเขียนใหม่ |
| Error Handling | Exceptions | Error codes |

### 1.2 การสร้าง PDO Connection

```php
<?php
// Database.php

class Database
{
    private static ?PDO $instance = null;
    
    private string $host;
    private string $dbname;
    private string $username;
    private string $password;
    private string $charset;
    
    private function __construct(array $config)
    {
        $this->host     = $config['host'] ?? 'localhost';
        $this->dbname   = $config['dbname'] ?? '';
        $this->username = $config['username'] ?? 'root';
        $this->password = $config['password'] ?? '';
        $this->charset  = $config['charset'] ?? 'utf8mb4';
    }
    
    /**
     * Singleton Pattern - ใช้ connection เดียว
     */
    public static function getInstance(array $config = []): PDO
    {
        if (self::$instance === null) {
            self::$instance = self::createConnection($config);
        }
        return self::$instance;
    }
    
    private static function createConnection(array $config): PDO
    {
        $host    = $config['host'] ?? 'localhost';
        $dbname  = $config['dbname'] ?? '';
        $charset = $config['charset'] ?? 'utf8mb4';
        
        $dsn = "mysql:host=$host;dbname=$dbname;charset=$charset";
        
        $options = [
            PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,    // throw exceptions
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,          // fetch as array
            PDO::ATTR_EMULATE_PREPARES   => false,                     // use real prepares
            PDO::MYSQL_ATTR_INIT_COMMAND => "SET NAMES $charset COLLATE utf8mb4_unicode_ci",
        ];
        
        try {
            $pdo = new PDO(
                $dsn,
                $config['username'] ?? 'root',
                $config['password'] ?? '',
                $options
            );
            return $pdo;
        } catch (PDOException $e) {
            // Log error แต่ไม่แสดงรายละเอียดให้ผู้ใช้
            error_log("Database connection failed: " . $e->getMessage());
            throw new RuntimeException("ไม่สามารถเชื่อมต่อฐานข้อมูลได้");
        }
    }
}

// config/database.php
$dbConfig = [
    'host'     => 'localhost',
    'dbname'   => 'blog_db',
    'username' => 'root',
    'password' => '',
    'charset'  => 'utf8mb4',
];

try {
    $pdo = Database::getInstance($dbConfig);
    echo "✅ เชื่อมต่อ Database สำเร็จ\n";
} catch (RuntimeException $e) {
    die("❌ " . $e->getMessage());
}
?>
```

### 1.3 DSN สำหรับ Database ต่างๆ

```php
<?php
// dsn-examples.php

// MySQL / MariaDB
$mysqlDsn = "mysql:host=localhost;port=3306;dbname=mydb;charset=utf8mb4";
$pdo = new PDO($mysqlDsn, 'user', 'pass');

// PostgreSQL
$pgDsn = "pgsql:host=localhost;port=5432;dbname=mydb";
$pdo = new PDO($pgDsn, 'user', 'pass');

// SQLite (ไม่ต้อง username/password)
$sqliteDsn = "sqlite:/path/to/database.sqlite";
$pdo = new PDO($sqliteDsn);

// SQLite ใน memory (สำหรับ testing)
$pdo = new PDO("sqlite::memory:");

// SQL Server
$mssqlDsn = "sqlsrv:Server=localhost;Database=mydb";
$pdo = new PDO($mssqlDsn, 'user', 'pass');
?>
```

---

## 📌 2. PDO CRUD

### 2.1 Query แบบต่างๆ

```php
<?php
// pdo-queries.php

// ======================================
// query() - สำหรับ queries ที่ไม่มี user input
// ======================================

$stmt = $pdo->query("SELECT * FROM categories");
$categories = $stmt->fetchAll(); // FETCH_ASSOC by default

foreach ($categories as $cat) {
    echo $cat['name'] . "\n";
}

// ======================================
// exec() - สำหรับ INSERT/UPDATE/DELETE ที่ไม่ return rows
// ======================================

$affected = $pdo->exec("UPDATE posts SET views = 0 WHERE status = 'draft'");
echo "อัปเดต $affected แถว\n";

// ======================================
// prepare() + execute() - ปลอดภัยที่สุด
// ======================================

// Positional Parameters (?)
$stmt = $pdo->prepare("SELECT * FROM posts WHERE status = ? AND user_id = ?");
$stmt->execute(['published', 1]);
$posts = $stmt->fetchAll();

// Named Parameters (:name) - อ่านง่ายกว่า
$stmt = $pdo->prepare("
    SELECT * FROM posts 
    WHERE status = :status AND user_id = :userId
    ORDER BY created_at DESC
    LIMIT :limit
");

$stmt->execute([
    ':status' => 'published',
    ':userId' => 1,
    ':limit'  => 10,
]);

$posts = $stmt->fetchAll();

// หรือ bindParam/bindValue
$status = 'published';
$userId = 1;
$limit = 10;

$stmt = $pdo->prepare("
    SELECT * FROM posts 
    WHERE status = :status AND user_id = :userId
    LIMIT :limit
");

// bindParam ผูกกับตัวแปร (reference) - เหมาะสำหรับ loop
$stmt->bindParam(':status', $status, PDO::PARAM_STR);
$stmt->bindParam(':userId', $userId, PDO::PARAM_INT);
$stmt->bindParam(':limit', $limit, PDO::PARAM_INT);

$stmt->execute();

// bindValue ผูกกับค่า (value)
$stmt->bindValue(':status', 'published', PDO::PARAM_STR);
$stmt->bindValue(':limit', 10, PDO::PARAM_INT);
?>
```

### 2.2 Fetch Methods

```php
<?php
// pdo-fetch.php

$stmt = $pdo->prepare("SELECT * FROM posts WHERE status = ?");
$stmt->execute(['published']);

// fetch() - อ่านทีละแถว
while ($row = $stmt->fetch()) {
    echo $row['title'] . "\n";
}

// fetchAll() - อ่านทั้งหมด
$stmt->execute(['published']);
$posts = $stmt->fetchAll();

// fetchAll(PDO::FETCH_CLASS) - อ่านเป็น Object
class Post
{
    public int $id;
    public string $title;
    public string $content;
    
    public function getExcerpt(int $length = 100): string
    {
        return mb_substr(strip_tags($this->content), 0, $length);
    }
}

$stmt->execute(['published']);
$posts = $stmt->fetchAll(PDO::FETCH_CLASS, Post::class);

foreach ($posts as $post) {
    echo $post->title . ": " . $post->getExcerpt() . "\n";
}

// fetchColumn() - อ่านคอลัมน์เดียว
$stmt = $pdo->prepare("SELECT COUNT(*) FROM posts WHERE status = ?");
$stmt->execute(['published']);
$count = $stmt->fetchColumn();
echo "จำนวนโพสต์: $count\n";

// fetchAll(PDO::FETCH_KEY_PAIR) - key => value
$stmt = $pdo->query("SELECT id, name FROM categories");
$categories = $stmt->fetchAll(PDO::FETCH_KEY_PAIR);
// result: [1 => 'เทคโนโลยี', 2 => 'การเดินทาง', ...]
?>
```

### 2.3 PDO CRUD ครบ

```php
<?php
// pdo-crud.php

class PostRepository
{
    private PDO $pdo;
    
    public function __construct(PDO $pdo)
    {
        $this->pdo = $pdo;
    }
    
    // ======================================
    // CREATE
    // ======================================
    
    public function create(array $data): int
    {
        $stmt = $this->pdo->prepare("
            INSERT INTO posts (title, slug, content, excerpt, user_id, category_id, status)
            VALUES (:title, :slug, :content, :excerpt, :userId, :categoryId, :status)
        ");
        
        $stmt->execute([
            ':title'      => $data['title'],
            ':slug'       => $data['slug'],
            ':content'    => $data['content'],
            ':excerpt'    => $data['excerpt'] ?? mb_substr(strip_tags($data['content']), 0, 200),
            ':userId'     => $data['user_id'],
            ':categoryId' => $data['category_id'] ?? null,
            ':status'     => $data['status'] ?? 'draft',
        ]);
        
        return (int)$this->pdo->lastInsertId();
    }
    
    // ======================================
    // READ
    // ======================================
    
    public function findById(int $id): ?array
    {
        $stmt = $this->pdo->prepare("
            SELECT p.*, u.name AS author, c.name AS category
            FROM posts p
            LEFT JOIN users u ON p.user_id = u.id
            LEFT JOIN categories c ON p.category_id = c.id
            WHERE p.id = :id
        ");
        
        $stmt->execute([':id' => $id]);
        $result = $stmt->fetch();
        
        return $result ?: null;
    }
    
    public function findBySlug(string $slug): ?array
    {
        $stmt = $this->pdo->prepare("
            SELECT p.*, u.name AS author, c.name AS category
            FROM posts p
            LEFT JOIN users u ON p.user_id = u.id
            LEFT JOIN categories c ON p.category_id = c.id
            WHERE p.slug = :slug AND p.status = 'published'
        ");
        
        $stmt->execute([':slug' => $slug]);
        return $stmt->fetch() ?: null;
    }
    
    public function getAll(array $options = []): array
    {
        $where = ['1=1'];
        $params = [];
        
        if (isset($options['status'])) {
            $where[] = 'p.status = :status';
            $params[':status'] = $options['status'];
        }
        
        if (isset($options['user_id'])) {
            $where[] = 'p.user_id = :userId';
            $params[':userId'] = $options['user_id'];
        }
        
        if (isset($options['category_id'])) {
            $where[] = 'p.category_id = :categoryId';
            $params[':categoryId'] = $options['category_id'];
        }
        
        if (isset($options['search'])) {
            $where[] = '(p.title LIKE :search OR p.content LIKE :search)';
            $params[':search'] = '%' . $options['search'] . '%';
        }
        
        $whereStr = implode(' AND ', $where);
        $order = $options['order'] ?? 'p.created_at DESC';
        $limit = isset($options['limit']) ? 'LIMIT :limit' : '';
        $offset = isset($options['offset']) ? 'OFFSET :offset' : '';
        
        if (isset($options['limit'])) $params[':limit'] = $options['limit'];
        if (isset($options['offset'])) $params[':offset'] = $options['offset'];
        
        $sql = "
            SELECT p.*, u.name AS author, c.name AS category
            FROM posts p
            LEFT JOIN users u ON p.user_id = u.id
            LEFT JOIN categories c ON p.category_id = c.id
            WHERE $whereStr
            ORDER BY $order
            $limit $offset
        ";
        
        $stmt = $this->pdo->prepare($sql);
        
        // Bind limit และ offset เป็น integer
        foreach ($params as $key => $value) {
            if (in_array($key, [':limit', ':offset'])) {
                $stmt->bindValue($key, (int)$value, PDO::PARAM_INT);
            } else {
                $stmt->bindValue($key, $value);
            }
        }
        
        $stmt->execute();
        return $stmt->fetchAll();
    }
    
    public function count(array $options = []): int
    {
        $where = ['1=1'];
        $params = [];
        
        if (isset($options['status'])) {
            $where[] = 'status = :status';
            $params[':status'] = $options['status'];
        }
        
        $whereStr = implode(' AND ', $where);
        $stmt = $this->pdo->prepare("SELECT COUNT(*) FROM posts WHERE $whereStr");
        $stmt->execute($params);
        
        return (int)$stmt->fetchColumn();
    }
    
    // ======================================
    // UPDATE
    // ======================================
    
    public function update(int $id, array $data): bool
    {
        $setClauses = [];
        $params = [':id' => $id];
        
        $allowedFields = ['title', 'content', 'excerpt', 'category_id', 'status'];
        
        foreach ($allowedFields as $field) {
            if (isset($data[$field])) {
                $setClauses[] = "$field = :$field";
                $params[":$field"] = $data[$field];
            }
        }
        
        if (empty($setClauses)) return false;
        
        // เพิ่ม updated_at
        $setClauses[] = 'updated_at = NOW()';
        
        $sql = "UPDATE posts SET " . implode(', ', $setClauses) . " WHERE id = :id";
        $stmt = $this->pdo->prepare($sql);
        $stmt->execute($params);
        
        return $stmt->rowCount() > 0;
    }
    
    // ======================================
    // DELETE
    // ======================================
    
    public function delete(int $id): bool
    {
        $stmt = $this->pdo->prepare("DELETE FROM posts WHERE id = :id");
        $stmt->execute([':id' => $id]);
        return $stmt->rowCount() > 0;
    }
    
    public function softDelete(int $id): bool
    {
        return $this->update($id, ['status' => 'archived']);
    }
    
    // ======================================
    // Statistics
    // ======================================
    
    public function getStats(): array
    {
        $stmt = $this->pdo->query("
            SELECT 
                status,
                COUNT(*) AS count,
                SUM(views) AS total_views,
                AVG(views) AS avg_views
            FROM posts
            GROUP BY status
        ");
        
        return $stmt->fetchAll();
    }
}

// ======================================
// ตัวอย่างการใช้งาน
// ======================================

$repo = new PostRepository($pdo);

// Create
$newId = $repo->create([
    'title'       => 'บทเรียน PHP PDO',
    'slug'        => 'php-pdo-tutorial',
    'content'     => 'เนื้อหาเกี่ยวกับ PDO...',
    'user_id'     => 1,
    'category_id' => 1,
    'status'      => 'draft',
]);

echo "สร้างโพสต์ ID: $newId\n";

// Read
$post = $repo->findById($newId);
echo "ชื่อ: " . $post['title'] . "\n";

// Update
$repo->update($newId, ['status' => 'published']);
echo "เผยแพร่แล้ว\n";

// List
$publishedPosts = $repo->getAll(['status' => 'published', 'limit' => 5]);
echo "โพสต์ที่เผยแพร่: " . count($publishedPosts) . " รายการ\n";

// Delete
$repo->softDelete($newId);
echo "Archive แล้ว\n";
?>
```

---

## 📌 3. PDO Transactions

```php
<?php
// pdo-transactions.php

class OrderService
{
    private PDO $pdo;
    
    public function __construct(PDO $pdo)
    {
        $this->pdo = $pdo;
    }
    
    /**
     * สร้างคำสั่งซื้อพร้อม items
     */
    public function createOrder(int $userId, array $items): int
    {
        // ตรวจสอบสต็อกก่อน
        foreach ($items as $item) {
            $stmt = $this->pdo->prepare("SELECT stock FROM products WHERE id = ?");
            $stmt->execute([$item['product_id']]);
            $product = $stmt->fetch();
            
            if (!$product || $product['stock'] < $item['quantity']) {
                throw new Exception("สินค้า ID {$item['product_id']} มีสต็อกไม่เพียงพอ");
            }
        }
        
        // เริ่ม Transaction
        $this->pdo->beginTransaction();
        
        try {
            // 1. สร้าง Order
            $total = array_sum(array_map(fn($i) => $i['price'] * $i['quantity'], $items));
            
            $stmt = $this->pdo->prepare("
                INSERT INTO orders (user_id, total, status, created_at)
                VALUES (:userId, :total, 'pending', NOW())
            ");
            $stmt->execute([':userId' => $userId, ':total' => $total]);
            $orderId = (int)$this->pdo->lastInsertId();
            
            // 2. เพิ่ม Order Items
            $itemStmt = $this->pdo->prepare("
                INSERT INTO order_items (order_id, product_id, quantity, price)
                VALUES (:orderId, :productId, :qty, :price)
            ");
            
            // 3. ลดสต็อก
            $stockStmt = $this->pdo->prepare("
                UPDATE products 
                SET stock = stock - :qty
                WHERE id = :productId AND stock >= :qty
            ");
            
            foreach ($items as $item) {
                // Insert order item
                $itemStmt->execute([
                    ':orderId'   => $orderId,
                    ':productId' => $item['product_id'],
                    ':qty'       => $item['quantity'],
                    ':price'     => $item['price'],
                ]);
                
                // ลดสต็อก
                $stockStmt->execute([
                    ':qty'       => $item['quantity'],
                    ':productId' => $item['product_id'],
                ]);
                
                if ($stockStmt->rowCount() === 0) {
                    throw new Exception("ไม่สามารถลดสต็อกสินค้า ID {$item['product_id']} ได้");
                }
            }
            
            // 4. บันทึก Log
            $logStmt = $this->pdo->prepare("
                INSERT INTO order_logs (order_id, action, created_at)
                VALUES (:orderId, 'created', NOW())
            ");
            $logStmt->execute([':orderId' => $orderId]);
            
            // Commit ถ้าทุกอย่างสำเร็จ
            $this->pdo->commit();
            
            return $orderId;
            
        } catch (Exception $e) {
            $this->pdo->rollback();
            throw $e;
        }
    }
    
    /**
     * Savepoint - Transaction ซ้อน Transaction
     */
    public function bulkProcess(array $orders): array
    {
        $results = [];
        $this->pdo->beginTransaction();
        
        foreach ($orders as $index => $order) {
            $savepoint = "sp_order_$index";
            $this->pdo->exec("SAVEPOINT $savepoint");
            
            try {
                $orderId = $this->processOrder($order);
                $results[] = ['success' => true, 'order_id' => $orderId];
                
            } catch (Exception $e) {
                // Rollback เฉพาะ order นี้ ไม่กระทบ order อื่น
                $this->pdo->exec("ROLLBACK TO SAVEPOINT $savepoint");
                $results[] = ['success' => false, 'error' => $e->getMessage()];
            }
        }
        
        $this->pdo->commit();
        return $results;
    }
    
    private function processOrder(array $order): int
    {
        $stmt = $this->pdo->prepare("INSERT INTO orders (user_id, total) VALUES (?, ?)");
        $stmt->execute([$order['user_id'], $order['total']]);
        return (int)$this->pdo->lastInsertId();
    }
}
?>
```

---

## 📌 4. Query Builder แบบง่าย

```php
<?php
// QueryBuilder.php

class QueryBuilder
{
    private PDO $pdo;
    private string $table = '';
    private array $wheres = [];
    private array $bindings = [];
    private array $selects = ['*'];
    private array $joins = [];
    private array $orders = [];
    private ?int $limitVal = null;
    private ?int $offsetVal = null;
    
    public function __construct(PDO $pdo)
    {
        $this->pdo = $pdo;
    }
    
    public function table(string $table): self
    {
        $clone = clone $this;
        $clone->table = $table;
        return $clone;
    }
    
    public function select(string ...$columns): self
    {
        $clone = clone $this;
        $clone->selects = $columns;
        return $clone;
    }
    
    public function where(string $column, mixed $valueOrOperator, mixed $value = null): self
    {
        $clone = clone $this;
        
        if ($value === null) {
            $operator = '=';
            $val = $valueOrOperator;
        } else {
            $operator = $valueOrOperator;
            $val = $value;
        }
        
        $placeholder = ':where_' . str_replace('.', '_', $column) . '_' . count($clone->wheres);
        $clone->wheres[] = "$column $operator $placeholder";
        $clone->bindings[$placeholder] = $val;
        
        return $clone;
    }
    
    public function whereIn(string $column, array $values): self
    {
        $clone = clone $this;
        $placeholders = [];
        
        foreach ($values as $i => $value) {
            $key = ":in_{$column}_{$i}";
            $placeholders[] = $key;
            $clone->bindings[$key] = $value;
        }
        
        $clone->wheres[] = "$column IN (" . implode(', ', $placeholders) . ")";
        
        return $clone;
    }
    
    public function whereNull(string $column): self
    {
        $clone = clone $this;
        $clone->wheres[] = "$column IS NULL";
        return $clone;
    }
    
    public function whereNotNull(string $column): self
    {
        $clone = clone $this;
        $clone->wheres[] = "$column IS NOT NULL";
        return $clone;
    }
    
    public function join(string $table, string $first, string $operator, string $second): self
    {
        $clone = clone $this;
        $clone->joins[] = "INNER JOIN $table ON $first $operator $second";
        return $clone;
    }
    
    public function leftJoin(string $table, string $first, string $operator, string $second): self
    {
        $clone = clone $this;
        $clone->joins[] = "LEFT JOIN $table ON $first $operator $second";
        return $clone;
    }
    
    public function orderBy(string $column, string $direction = 'ASC'): self
    {
        $clone = clone $this;
        $clone->orders[] = "$column $direction";
        return $clone;
    }
    
    public function limit(int $limit): self
    {
        $clone = clone $this;
        $clone->limitVal = $limit;
        return $clone;
    }
    
    public function offset(int $offset): self
    {
        $clone = clone $this;
        $clone->offsetVal = $offset;
        return $clone;
    }
    
    public function page(int $page, int $perPage = 15): self
    {
        return $this->limit($perPage)->offset(($page - 1) * $perPage);
    }
    
    private function buildSelect(): string
    {
        $columns = implode(', ', $this->selects);
        $from = "FROM {$this->table}";
        $joins = !empty($this->joins) ? implode(' ', $this->joins) : '';
        $where = !empty($this->wheres) ? 'WHERE ' . implode(' AND ', $this->wheres) : '';
        $order = !empty($this->orders) ? 'ORDER BY ' . implode(', ', $this->orders) : '';
        $limit = $this->limitVal !== null ? "LIMIT {$this->limitVal}" : '';
        $offset = $this->offsetVal !== null ? "OFFSET {$this->offsetVal}" : '';
        
        return trim("SELECT $columns $from $joins $where $order $limit $offset");
    }
    
    /**
     * ดึงข้อมูลทั้งหมด
     */
    public function get(): array
    {
        $stmt = $this->pdo->prepare($this->buildSelect());
        $stmt->execute($this->bindings);
        return $stmt->fetchAll();
    }
    
    /**
     * ดึงแถวแรก
     */
    public function first(): ?array
    {
        $result = $this->limit(1)->get();
        return $result[0] ?? null;
    }
    
    /**
     * นับจำนวน
     */
    public function count(): int
    {
        $from = "FROM {$this->table}";
        $joins = !empty($this->joins) ? implode(' ', $this->joins) : '';
        $where = !empty($this->wheres) ? 'WHERE ' . implode(' AND ', $this->wheres) : '';
        
        $sql = trim("SELECT COUNT(*) $from $joins $where");
        $stmt = $this->pdo->prepare($sql);
        $stmt->execute($this->bindings);
        return (int)$stmt->fetchColumn();
    }
    
    /**
     * INSERT
     */
    public function insert(array $data): int
    {
        $columns = implode(', ', array_keys($data));
        $placeholders = implode(', ', array_map(fn($k) => ":$k", array_keys($data)));
        
        $params = [];
        foreach ($data as $key => $value) {
            $params[":$key"] = $value;
        }
        
        $stmt = $this->pdo->prepare("INSERT INTO {$this->table} ($columns) VALUES ($placeholders)");
        $stmt->execute($params);
        
        return (int)$this->pdo->lastInsertId();
    }
    
    /**
     * UPDATE
     */
    public function update(array $data): int
    {
        $setClauses = array_map(fn($k) => "$k = :set_$k", array_keys($data));
        $setStr = implode(', ', $setClauses);
        
        $params = $this->bindings;
        foreach ($data as $key => $value) {
            $params[":set_$key"] = $value;
        }
        
        $where = !empty($this->wheres) ? 'WHERE ' . implode(' AND ', $this->wheres) : '';
        $sql = "UPDATE {$this->table} SET $setStr $where";
        
        $stmt = $this->pdo->prepare($sql);
        $stmt->execute($params);
        
        return $stmt->rowCount();
    }
    
    /**
     * DELETE
     */
    public function delete(): int
    {
        $where = !empty($this->wheres) ? 'WHERE ' . implode(' AND ', $this->wheres) : '';
        $sql = "DELETE FROM {$this->table} $where";
        
        $stmt = $this->pdo->prepare($sql);
        $stmt->execute($this->bindings);
        
        return $stmt->rowCount();
    }
    
    /**
     * Paginate
     */
    public function paginate(int $perPage = 15, int $page = 1): array
    {
        $total = $this->count();
        $data = $this->page($page, $perPage)->get();
        
        return [
            'data'         => $data,
            'total'        => $total,
            'per_page'     => $perPage,
            'current_page' => $page,
            'last_page'    => (int)ceil($total / $perPage),
            'from'         => ($page - 1) * $perPage + 1,
            'to'           => min($page * $perPage, $total),
        ];
    }
}

// ======================================
// ตัวอย่างการใช้งาน Query Builder
// ======================================

$qb = new QueryBuilder($pdo);

// SELECT แบบง่าย
$posts = $qb->table('posts')
    ->where('status', 'published')
    ->orderBy('created_at', 'DESC')
    ->limit(10)
    ->get();

// SELECT พร้อม JOIN และ conditions
$posts = $qb->table('posts as p')
    ->select('p.id', 'p.title', 'p.created_at', 'u.name as author', 'c.name as category')
    ->leftJoin('users u', 'p.user_id', '=', 'u.id')
    ->leftJoin('categories c', 'p.category_id', '=', 'c.id')
    ->where('p.status', 'published')
    ->where('p.views', '>', 100)
    ->orderBy('p.created_at', 'DESC')
    ->page(1, 10)
    ->get();

// นับ
$count = $qb->table('posts')->where('status', 'published')->count();

// INSERT
$newId = $qb->table('posts')->insert([
    'title'   => 'New Post',
    'content' => 'Content here',
    'user_id' => 1,
    'status'  => 'draft',
]);

// UPDATE
$affected = $qb->table('posts')
    ->where('id', $newId)
    ->update(['status' => 'published']);

// WHERE IN
$activePosts = $qb->table('posts')
    ->whereIn('status', ['published', 'draft'])
    ->get();

// Paginate
$result = $qb->table('posts')
    ->where('status', 'published')
    ->orderBy('created_at', 'DESC')
    ->paginate(10, 1);

echo "หน้า {$result['current_page']} จาก {$result['last_page']}\n";
echo "แสดง {$result['from']}-{$result['to']} จาก {$result['total']} รายการ\n";
?>
```

---

## 🛠️ Workshop: Data Access Layer

```php
<?php
// Model.php - Base Model

abstract class Model
{
    protected PDO $pdo;
    protected string $table;
    protected string $primaryKey = 'id';
    protected array $fillable = [];
    protected array $hidden = ['password'];
    
    public function __construct(PDO $pdo)
    {
        $this->pdo = $pdo;
    }
    
    protected function query(): QueryBuilder
    {
        return (new QueryBuilder($this->pdo))->table($this->table);
    }
    
    public function all(): array
    {
        return $this->query()->get();
    }
    
    public function find(int $id): ?array
    {
        $result = $this->query()->where($this->primaryKey, $id)->first();
        return $result ? $this->hideFields($result) : null;
    }
    
    public function create(array $data): int
    {
        $filtered = $this->filterFillable($data);
        return $this->query()->insert($filtered);
    }
    
    public function update(int $id, array $data): bool
    {
        $filtered = $this->filterFillable($data);
        $affected = $this->query()->where($this->primaryKey, $id)->update($filtered);
        return $affected > 0;
    }
    
    public function delete(int $id): bool
    {
        $affected = $this->query()->where($this->primaryKey, $id)->delete();
        return $affected > 0;
    }
    
    public function paginate(int $perPage = 15, int $page = 1): array
    {
        return $this->query()->paginate($perPage, $page);
    }
    
    private function filterFillable(array $data): array
    {
        if (empty($this->fillable)) return $data;
        return array_intersect_key($data, array_flip($this->fillable));
    }
    
    private function hideFields(array $data): array
    {
        foreach ($this->hidden as $field) {
            unset($data[$field]);
        }
        return $data;
    }
}

// UserModel.php
class UserModel extends Model
{
    protected string $table = 'users';
    protected array $fillable = ['username', 'email', 'password', 'name', 'role'];
    protected array $hidden = ['password'];
    
    public function findByEmail(string $email): ?array
    {
        return $this->query()->where('email', $email)->first();
    }
    
    public function findByUsername(string $username): ?array
    {
        return $this->query()->where('username', $username)->first();
    }
    
    public function withPosts(int $userId): ?array
    {
        $user = $this->find($userId);
        if (!$user) return null;
        
        $postModel = new PostModel($this->pdo);
        $user['posts'] = $postModel->getByUser($userId);
        
        return $user;
    }
}

// PostModel.php
class PostModel extends Model
{
    protected string $table = 'posts';
    protected array $fillable = ['title', 'slug', 'content', 'excerpt', 'user_id', 'category_id', 'status'];
    
    public function getPublished(int $limit = 10, int $page = 1): array
    {
        return $this->query()
            ->where('status', 'published')
            ->orderBy('published_at', 'DESC')
            ->paginate($limit, $page);
    }
    
    public function getByUser(int $userId): array
    {
        return $this->query()->where('user_id', $userId)->get();
    }
    
    public function findBySlug(string $slug): ?array
    {
        return $this->query()
            ->where('slug', $slug)
            ->where('status', 'published')
            ->first();
    }
    
    public function search(string $query, int $limit = 10): array
    {
        $stmt = $this->pdo->prepare("
            SELECT * FROM posts 
            WHERE status = 'published' 
            AND (title LIKE :q OR content LIKE :q)
            ORDER BY created_at DESC
            LIMIT :limit
        ");
        
        $stmt->bindValue(':q', '%' . $query . '%');
        $stmt->bindValue(':limit', $limit, PDO::PARAM_INT);
        $stmt->execute();
        
        return $stmt->fetchAll();
    }
}

// ======================================
// Repository Pattern
// ======================================

interface PostRepositoryInterface
{
    public function getPublished(int $limit, int $page): array;
    public function findById(int $id): ?array;
    public function create(array $data): int;
    public function update(int $id, array $data): bool;
    public function delete(int $id): bool;
}

class PdoPostRepository implements PostRepositoryInterface
{
    private PostModel $model;
    
    public function __construct(PDO $pdo)
    {
        $this->model = new PostModel($pdo);
    }
    
    public function getPublished(int $limit = 10, int $page = 1): array
    {
        return $this->model->getPublished($limit, $page);
    }
    
    public function findById(int $id): ?array
    {
        return $this->model->find($id);
    }
    
    public function create(array $data): int
    {
        return $this->model->create($data);
    }
    
    public function update(int $id, array $data): bool
    {
        return $this->model->update($id, $data);
    }
    
    public function delete(int $id): bool
    {
        return $this->model->delete($id);
    }
}

// ตัวอย่างการใช้งาน
$userModel = new UserModel($pdo);
$postModel = new PostModel($pdo);

// สร้าง user
$userId = $userModel->create([
    'username' => 'johndoe',
    'email'    => 'john@example.com',
    'password' => password_hash('Secret123', PASSWORD_BCRYPT),
    'name'     => 'John Doe',
    'role'     => 'author',
]);

// สร้าง post
$postId = $postModel->create([
    'title'   => 'ทดสอบ PDO',
    'slug'    => 'test-pdo',
    'content' => 'เนื้อหาทดสอบ PDO...',
    'user_id' => $userId,
    'status'  => 'published',
]);

// ดึงข้อมูล
$posts = $postModel->getPublished(5, 1);
echo "โพสต์: " . count($posts['data']) . " รายการ (ทั้งหมด {$posts['total']})\n";

// ค้นหา
$results = $postModel->search('PDO');
echo "ผลการค้นหา: " . count($results) . " รายการ\n";
?>
```

---

## 📝 Quiz

### คำถาม

**ข้อ 1:** ข้อใดคือข้อดีหลักของ PDO เมื่อเทียบกับ MySQLi?
- A. PDO เร็วกว่า MySQLi มาก
- B. PDO รองรับหลาย Database เช่น MySQL, PostgreSQL, SQLite
- C. PDO ใช้น้อย memory กว่า
- D. PDO มี built-in Query Builder

**ข้อ 2:** `PDO::ERRMODE_EXCEPTION` ทำอะไร?
- A. ปิดการแสดง error
- B. throw PDOException เมื่อเกิด error
- C. บันทึก error ลง log file
- D. ส่ง error code เท่านั้น

**ข้อ 3:** ความแตกต่างระหว่าง `bindParam()` และ `bindValue()`?
- A. `bindParam` ผูกกับตัวแปร (reference), `bindValue` ผูกกับค่า
- B. `bindParam` ใช้ได้กับ string, `bindValue` ใช้ได้กับ integer
- C. `bindValue` ปลอดภัยกว่า `bindParam`
- D. ไม่มีความแตกต่าง

**ข้อ 4:** `$pdo->lastInsertId()` ต้องเรียกเมื่อใด?
- A. ก่อน execute INSERT
- B. หลัง execute INSERT
- C. ก่อน beginTransaction
- D. หลัง commit

**ข้อ 5:** Repository Pattern ช่วยอะไร?
- A. ทำให้ query เร็วขึ้น
- B. แยก Database logic ออกจาก Business logic
- C. ป้องกัน SQL Injection
- D. ลด memory usage

### เฉลย

| ข้อ | คำตอบ | อธิบาย |
|-----|-------|--------|
| 1 | B | PDO รองรับ 12+ database drivers ทำให้เปลี่ยน database ได้โดยแก้แค่ DSN |
| 2 | B | ERRMODE_EXCEPTION ทำให้ PHP throw PDOException แทนที่จะ silent fail |
| 3 | A | `bindParam` ผูก reference ของตัวแปร ส่วน `bindValue` คัดลอกค่า ณ เวลานั้น |
| 4 | B | `lastInsertId()` return AUTO_INCREMENT ID ของแถวที่เพิ่งถูก INSERT |
| 5 | B | Repository Pattern ทำให้ code ยืดหยุ่น testable และเปลี่ยน database ได้ง่าย |

---

## ➡️ Part ถัดไป

**[Part 14: PHP OOP Basics - พื้นฐาน Object-Oriented Programming](part-014-php-oop-basics.md)**
