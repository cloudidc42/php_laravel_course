# 🗄️ Part 12: PHP MySQL - การทำงานกับฐานข้อมูล MySQL

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เชื่อมต่อกับ MySQL ด้วย PHP ได้
- ทำ CRUD operations (Create, Read, Update, Delete) ได้
- ใช้ Prepared Statements เพื่อป้องกัน SQL Injection ได้
- ใช้ Transactions ได้
- สร้าง Blog CRUD System ได้

---

## 📌 1. MySQL Connection

### 1.1 ติดตั้งและตั้งค่า MySQL

```sql
-- สร้าง Database
CREATE DATABASE IF NOT EXISTS blog_db
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

USE blog_db;

-- สร้างตาราง users
CREATE TABLE users (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    username    VARCHAR(50) UNIQUE NOT NULL,
    email       VARCHAR(100) UNIQUE NOT NULL,
    password    VARCHAR(255) NOT NULL,
    name        VARCHAR(100),
    role        ENUM('admin', 'author', 'user') DEFAULT 'user',
    avatar      VARCHAR(255),
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- สร้างตาราง categories
CREATE TABLE categories (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    slug        VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- สร้างตาราง posts
CREATE TABLE posts (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    title       VARCHAR(255) NOT NULL,
    slug        VARCHAR(255) UNIQUE NOT NULL,
    content     LONGTEXT NOT NULL,
    excerpt     TEXT,
    user_id     INT NOT NULL,
    category_id INT,
    status      ENUM('draft', 'published', 'archived') DEFAULT 'draft',
    views       INT DEFAULT 0,
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    published_at DATETIME,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE SET NULL,
    INDEX idx_status (status),
    INDEX idx_user_id (user_id),
    INDEX idx_category_id (category_id)
);

-- สร้างตาราง comments
CREATE TABLE comments (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    post_id     INT NOT NULL,
    user_id     INT,
    author_name VARCHAR(100),
    author_email VARCHAR(100),
    content     TEXT NOT NULL,
    status      ENUM('pending', 'approved', 'spam') DEFAULT 'pending',
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (post_id) REFERENCES posts(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL
);

-- Insert ข้อมูลตัวอย่าง
INSERT INTO categories (name, slug, description) VALUES
('เทคโนโลยี', 'technology', 'บทความเกี่ยวกับเทคโนโลยีและนวัตกรรม'),
('การเดินทาง', 'travel', 'บทความท่องเที่ยวทั่วโลก'),
('อาหาร', 'food', 'สูตรอาหารและรีวิวร้านอาหาร'),
('ไลฟ์สไตล์', 'lifestyle', 'เคล็ดลับการใช้ชีวิต');

INSERT INTO users (username, email, password, name, role) VALUES
('admin', 'admin@blog.com', '$2y$10$example_hash_here', 'ผู้ดูแลระบบ', 'admin'),
('author1', 'author@blog.com', '$2y$10$example_hash_here', 'นักเขียน A', 'author');
```

### 1.2 MySQLi Connection

```php
<?php
// config/database.php

// ======================================
// วิธีที่ 1: MySQLi Procedural
// ======================================
$host = 'localhost';
$dbname = 'blog_db';
$username = 'root';
$password = '';
$charset = 'utf8mb4';

// เชื่อมต่อ
$conn = mysqli_connect($host, $username, $password, $dbname);

// ตรวจสอบการเชื่อมต่อ
if (!$conn) {
    die("Connection failed: " . mysqli_connect_error());
}

// ตั้งค่า charset
mysqli_set_charset($conn, $charset);

echo "Connected successfully!";

// ======================================
// วิธีที่ 2: MySQLi OOP
// ======================================
$mysqli = new mysqli($host, $username, $password, $dbname);

if ($mysqli->connect_error) {
    die("Connection failed: " . $mysqli->connect_error);
}

$mysqli->set_charset($charset);

// ======================================
// วิธีที่ 3: Error Handling ที่ดีกว่า
// ======================================
mysqli_report(MYSQLI_REPORT_ERROR | MYSQLI_REPORT_STRICT);

try {
    $mysqli = new mysqli($host, $username, $password, $dbname);
    $mysqli->set_charset($charset);
    echo "✅ เชื่อมต่อ Database สำเร็จ";
} catch (mysqli_sql_exception $e) {
    error_log("Database connection failed: " . $e->getMessage());
    die("ไม่สามารถเชื่อมต่อฐานข้อมูลได้ กรุณาลองใหม่อีกครั้ง");
}
?>
```

---

## 📌 2. CRUD Operations

### 2.1 Create (INSERT)

```php
<?php
// mysqli-crud.php

// ======================================
// INSERT - เพิ่มข้อมูล
// ======================================

// วิธีที่ 1: Simple INSERT (ไม่ปลอดภัย - อย่าใช้กับ user input!)
$title = "PHP MySQL Tutorial";
$query = "INSERT INTO posts (title, content, user_id) VALUES ('$title', 'content here', 1)";
// ห้ามใช้แบบนี้กับข้อมูลจากผู้ใช้ เพราะเสี่ยง SQL Injection!

// วิธีที่ 2: Prepared Statement (แนะนำ)
$title = $_POST['title'] ?? 'Test Post';
$content = $_POST['content'] ?? 'Test Content';
$userId = 1;
$slug = createSlug($title);

$stmt = $mysqli->prepare("
    INSERT INTO posts (title, slug, content, user_id, status)
    VALUES (?, ?, ?, ?, 'draft')
");

// Bind parameters: s=string, i=integer, d=double, b=blob
$stmt->bind_param('sssi', $title, $slug, $content, $userId);
$stmt->execute();

$newId = $mysqli->insert_id; // ID ของแถวที่เพิ่ง insert
$affectedRows = $stmt->affected_rows;

echo "สร้างโพสต์สำเร็จ! ID: $newId\n";

$stmt->close();

// ======================================
// INSERT multiple rows
// ======================================
$posts = [
    ['PHP Basics', 'เรียน PHP เบื้องต้น', 1],
    ['Laravel Tutorial', 'เรียน Laravel', 1],
    ['MySQL Guide', 'คู่มือ MySQL', 2],
];

$stmt = $mysqli->prepare("INSERT INTO posts (title, content, user_id) VALUES (?, ?, ?)");

foreach ($posts as $post) {
    $stmt->bind_param('ssi', $post[0], $post[1], $post[2]);
    $stmt->execute();
    echo "เพิ่มโพสต์: {$post[0]} (ID: {$mysqli->insert_id})\n";
}

$stmt->close();

// Helper function
function createSlug(string $text): string
{
    // แปลงเป็น lowercase
    $text = strtolower($text);
    // แทนที่ช่องว่างด้วย -
    $text = preg_replace('/[\s\-]+/', '-', $text);
    // ลบ special characters
    $text = preg_replace('/[^a-z0-9\-]/', '', $text);
    // ลบ - ที่ขึ้นต้นและท้าย
    return trim($text, '-');
}
?>
```

### 2.2 Read (SELECT)

```php
<?php
// mysqli-select.php

// ======================================
// SELECT - อ่านข้อมูล
// ======================================

// SELECT ทั้งหมด
$result = $mysqli->query("SELECT * FROM posts WHERE status = 'published'");

while ($row = $result->fetch_assoc()) {
    echo $row['title'] . "\n";
}

$result->free();

// ======================================
// SELECT พร้อม Prepared Statement
// ======================================

// SELECT หนึ่ง record
$postId = 1;
$stmt = $mysqli->prepare("SELECT * FROM posts WHERE id = ?");
$stmt->bind_param('i', $postId);
$stmt->execute();

$result = $stmt->get_result();
$post = $result->fetch_assoc();

if ($post) {
    echo "ชื่อโพสต์: " . $post['title'] . "\n";
    echo "เนื้อหา: " . $post['content'] . "\n";
} else {
    echo "ไม่พบโพสต์\n";
}

$stmt->close();

// ======================================
// SELECT พร้อม JOIN
// ======================================

$stmt = $mysqli->prepare("
    SELECT 
        p.id,
        p.title,
        p.created_at,
        p.views,
        u.name AS author_name,
        c.name AS category_name,
        COUNT(cm.id) AS comment_count
    FROM posts p
    LEFT JOIN users u ON p.user_id = u.id
    LEFT JOIN categories c ON p.category_id = c.id
    LEFT JOIN comments cm ON p.id = cm.post_id AND cm.status = 'approved'
    WHERE p.status = ?
    GROUP BY p.id
    ORDER BY p.created_at DESC
    LIMIT ? OFFSET ?
");

$status = 'published';
$limit = 10;
$offset = 0;

$stmt->bind_param('sii', $status, $limit, $offset);
$stmt->execute();

$result = $stmt->get_result();

echo "โพสต์ที่เผยแพร่แล้ว:\n";
while ($post = $result->fetch_assoc()) {
    printf(
        "- %s (โดย %s | หมวด: %s | %d ความเห็น | %d views)\n",
        $post['title'],
        $post['author_name'],
        $post['category_name'] ?? 'ไม่มีหมวด',
        $post['comment_count'],
        $post['views']
    );
}

$stmt->close();

// ======================================
// COUNT, SUM, AVG
// ======================================

$stmt = $mysqli->prepare("
    SELECT 
        COUNT(*) AS total_posts,
        SUM(views) AS total_views,
        AVG(views) AS avg_views
    FROM posts
    WHERE user_id = ?
");

$userId = 1;
$stmt->bind_param('i', $userId);
$stmt->execute();

$stats = $stmt->get_result()->fetch_assoc();

echo sprintf(
    "โพสต์ทั้งหมด: %d | Views รวม: %d | Views เฉลี่ย: %.1f\n",
    $stats['total_posts'],
    $stats['total_views'],
    $stats['avg_views']
);

$stmt->close();

// ======================================
// SEARCH
// ======================================

$searchTerm = '%php%';
$stmt = $mysqli->prepare("
    SELECT id, title, created_at
    FROM posts
    WHERE (title LIKE ? OR content LIKE ?)
    AND status = 'published'
    ORDER BY created_at DESC
");

$stmt->bind_param('ss', $searchTerm, $searchTerm);
$stmt->execute();

$result = $stmt->get_result();
$posts = $result->fetch_all(MYSQLI_ASSOC);

echo "ผลการค้นหา: " . count($posts) . " รายการ\n";

$stmt->close();
?>
```

### 2.3 Update (UPDATE)

```php
<?php
// mysqli-update.php

// ======================================
// UPDATE - แก้ไขข้อมูล
// ======================================

// อัปเดตโพสต์
$postId = 1;
$title = "PHP MySQL Tutorial (Updated)";
$content = "เนื้อหาที่แก้ไขแล้ว";
$status = "published";
$publishedAt = date('Y-m-d H:i:s');

$stmt = $mysqli->prepare("
    UPDATE posts 
    SET title = ?, content = ?, status = ?, published_at = ?
    WHERE id = ?
");

$stmt->bind_param('ssssi', $title, $content, $status, $publishedAt, $postId);
$stmt->execute();

if ($stmt->affected_rows > 0) {
    echo "✅ อัปเดตสำเร็จ! แก้ไข {$stmt->affected_rows} แถว\n";
} else {
    echo "⚠️ ไม่มีแถวที่ถูกแก้ไข (อาจ ID ไม่มีหรือข้อมูลเหมือนเดิม)\n";
}

$stmt->close();

// ======================================
// เพิ่มค่า (Increment)
// ======================================

$postId = 1;
$stmt = $mysqli->prepare("UPDATE posts SET views = views + 1 WHERE id = ?");
$stmt->bind_param('i', $postId);
$stmt->execute();
$stmt->close();

// ======================================
// Conditional UPDATE
// ======================================

// อนุมัติ comments ที่รอ pending
$stmt = $mysqli->prepare("
    UPDATE comments 
    SET status = 'approved'
    WHERE post_id = ? AND status = 'pending'
");
$stmt->bind_param('i', $postId);
$stmt->execute();
echo "อนุมัติ {$stmt->affected_rows} ความเห็น\n";
$stmt->close();
?>
```

### 2.4 Delete (DELETE)

```php
<?php
// mysqli-delete.php

// ======================================
// DELETE - ลบข้อมูล
// ======================================

// ลบโพสต์
$postId = 5;
$stmt = $mysqli->prepare("DELETE FROM posts WHERE id = ?");
$stmt->bind_param('i', $postId);
$stmt->execute();

if ($stmt->affected_rows > 0) {
    echo "✅ ลบโพสต์ ID $postId สำเร็จ\n";
} else {
    echo "❌ ไม่พบโพสต์ ID $postId\n";
}

$stmt->close();

// ======================================
// Soft Delete (แนะนำ)
// ======================================

// แทนที่จะลบจริง ให้เปลี่ยน status เป็น archived
$stmt = $mysqli->prepare("
    UPDATE posts SET status = 'archived', updated_at = NOW()
    WHERE id = ? AND status != 'archived'
");
$stmt->bind_param('i', $postId);
$stmt->execute();
echo "Archive โพสต์สำเร็จ\n";
$stmt->close();

// ======================================
// DELETE หลายแถว
// ======================================

// ลบ spam comments
$stmt = $mysqli->prepare("DELETE FROM comments WHERE status = 'spam' AND created_at < ?");
$cutoffDate = date('Y-m-d', strtotime('-30 days')); // 30 วันที่แล้ว
$stmt->bind_param('s', $cutoffDate);
$stmt->execute();
echo "ลบ spam ที่เก่ากว่า 30 วัน: {$stmt->affected_rows} รายการ\n";
$stmt->close();
?>
```

---

## 📌 3. Prepared Statements

### 3.1 ทำไมต้องใช้ Prepared Statements?

```php
<?php
// sql-injection-demo.php

// ======================================
// SQL Injection Attack ตัวอย่าง
// ======================================

// โค้ดที่ไม่ปลอดภัย (อย่าทำแบบนี้!)
$username = $_POST['username'] ?? '';
$password = $_POST['password'] ?? '';

$query = "SELECT * FROM users WHERE username = '$username' AND password = '$password'";

// ถ้า attacker กรอก username = ' OR '1'='1' --
// Query จะกลายเป็น:
// SELECT * FROM users WHERE username = '' OR '1'='1' -- ' AND password = ''
// ซึ่งจะ return rows ทั้งหมดเพราะ '1'='1' เป็น true เสมอ!

// ======================================
// Prepared Statements ป้องกัน SQL Injection
// ======================================

$stmt = $mysqli->prepare("
    SELECT id, username, name, role 
    FROM users 
    WHERE username = ? AND password = ?
");

// ? จะถูก escape อัตโนมัติ
$stmt->bind_param('ss', $username, $password);
$stmt->execute();
$result = $stmt->get_result();
$user = $result->fetch_assoc();

if ($user) {
    echo "Login สำเร็จ! ยินดีต้อนรับ " . $user['name'];
} else {
    echo "ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง";
}

$stmt->close();
?>
```

### 3.2 Advanced Prepared Statements

```php
<?php
// advanced-prepared.php

class QueryBuilder
{
    private mysqli $db;
    
    public function __construct(mysqli $db)
    {
        $this->db = $db;
    }
    
    /**
     * SELECT ที่ยืดหยุ่น
     */
    public function select(
        string $table,
        array $conditions = [],
        array $options = []
    ): array {
        $where = '';
        $params = [];
        $types = '';
        
        if (!empty($conditions)) {
            $clauses = [];
            foreach ($conditions as $column => $value) {
                $clauses[] = "`$column` = ?";
                $params[] = $value;
                $types .= is_int($value) ? 'i' : 's';
            }
            $where = 'WHERE ' . implode(' AND ', $clauses);
        }
        
        $orderBy = isset($options['order']) ? "ORDER BY {$options['order']}" : '';
        $limit = isset($options['limit']) ? "LIMIT {$options['limit']}" : '';
        $offset = isset($options['offset']) ? "OFFSET {$options['offset']}" : '';
        
        $columns = $options['columns'] ?? '*';
        
        $sql = "SELECT $columns FROM `$table` $where $orderBy $limit $offset";
        
        $stmt = $this->db->prepare($sql);
        
        if (!empty($params)) {
            $stmt->bind_param($types, ...$params);
        }
        
        $stmt->execute();
        return $stmt->get_result()->fetch_all(MYSQLI_ASSOC);
    }
    
    /**
     * INSERT ที่ยืดหยุ่น
     */
    public function insert(string $table, array $data): int
    {
        $columns = implode(', ', array_keys($data));
        $placeholders = implode(', ', array_fill(0, count($data), '?'));
        $types = '';
        
        foreach ($data as $value) {
            if (is_int($value)) $types .= 'i';
            elseif (is_float($value)) $types .= 'd';
            else $types .= 's';
        }
        
        $sql = "INSERT INTO `$table` ($columns) VALUES ($placeholders)";
        $stmt = $this->db->prepare($sql);
        $stmt->bind_param($types, ...array_values($data));
        $stmt->execute();
        
        return $this->db->insert_id;
    }
    
    /**
     * UPDATE ที่ยืดหยุ่น
     */
    public function update(string $table, array $data, array $conditions): int
    {
        $setClauses = [];
        $params = [];
        $types = '';
        
        foreach ($data as $column => $value) {
            $setClauses[] = "`$column` = ?";
            $params[] = $value;
            $types .= is_int($value) ? 'i' : 's';
        }
        
        $whereClauses = [];
        foreach ($conditions as $column => $value) {
            $whereClauses[] = "`$column` = ?";
            $params[] = $value;
            $types .= is_int($value) ? 'i' : 's';
        }
        
        $sql = "UPDATE `$table` SET " . implode(', ', $setClauses) . " WHERE " . implode(' AND ', $whereClauses);
        $stmt = $this->db->prepare($sql);
        $stmt->bind_param($types, ...$params);
        $stmt->execute();
        
        return $stmt->affected_rows;
    }
}

// ตัวอย่างการใช้งาน
$qb = new QueryBuilder($mysqli);

// SELECT
$posts = $qb->select('posts', ['status' => 'published'], ['limit' => 10, 'order' => 'created_at DESC']);

// INSERT
$newId = $qb->insert('categories', ['name' => 'Programming', 'slug' => 'programming', 'description' => 'บทความเขียนโปรแกรม']);

// UPDATE
$affected = $qb->update('posts', ['views' => 100], ['id' => 1]);
?>
```

---

## 📌 4. Transactions

Transaction คือกลุ่มของ queries ที่ต้องทำงานสำเร็จทั้งหมด หรือ rollback ทั้งหมด (All or Nothing)

```php
<?php
// transactions.php

// ======================================
// ตัวอย่าง Transaction: โอนเงิน
// ======================================

function transferMoney(mysqli $db, int $fromUserId, int $toUserId, float $amount): bool
{
    $db->begin_transaction();
    
    try {
        // 1. ตรวจสอบยอดเงิน
        $stmt = $db->prepare("SELECT balance FROM wallets WHERE user_id = ? FOR UPDATE");
        $stmt->bind_param('i', $fromUserId);
        $stmt->execute();
        $fromWallet = $stmt->get_result()->fetch_assoc();
        $stmt->close();
        
        if (!$fromWallet || $fromWallet['balance'] < $amount) {
            throw new Exception("ยอดเงินไม่เพียงพอ");
        }
        
        // 2. หักเงินจากบัญชีต้นทาง
        $stmt = $db->prepare("UPDATE wallets SET balance = balance - ? WHERE user_id = ?");
        $stmt->bind_param('di', $amount, $fromUserId);
        $stmt->execute();
        
        if ($stmt->affected_rows !== 1) {
            throw new Exception("ไม่สามารถหักเงินได้");
        }
        $stmt->close();
        
        // 3. เพิ่มเงินในบัญชีปลายทาง
        $stmt = $db->prepare("UPDATE wallets SET balance = balance + ? WHERE user_id = ?");
        $stmt->bind_param('di', $amount, $toUserId);
        $stmt->execute();
        
        if ($stmt->affected_rows !== 1) {
            throw new Exception("ไม่สามารถเพิ่มเงินได้");
        }
        $stmt->close();
        
        // 4. บันทึก transaction log
        $stmt = $db->prepare("
            INSERT INTO transactions (from_user_id, to_user_id, amount, type, created_at)
            VALUES (?, ?, ?, 'transfer', NOW())
        ");
        $stmt->bind_param('iid', $fromUserId, $toUserId, $amount);
        $stmt->execute();
        $stmt->close();
        
        // 5. Commit ถ้าทุกอย่างสำเร็จ
        $db->commit();
        return true;
        
    } catch (Exception $e) {
        // Rollback ถ้ามีข้อผิดพลาด
        $db->rollback();
        error_log("Transfer failed: " . $e->getMessage());
        throw $e; // re-throw เพื่อให้ caller จัดการ
    }
}

// ตัวอย่างการใช้งาน
try {
    transferMoney($mysqli, 1, 2, 1000.00);
    echo "✅ โอนเงิน 1,000 บาท สำเร็จ!\n";
} catch (Exception $e) {
    echo "❌ โอนเงินไม่สำเร็จ: " . $e->getMessage() . "\n";
}

// ======================================
// Transaction สำหรับ Blog Post
// ======================================

function publishPostWithNotification(mysqli $db, int $postId, array $tagIds): bool
{
    $db->begin_transaction();
    
    try {
        // 1. เปลี่ยน status เป็น published
        $stmt = $db->prepare("
            UPDATE posts 
            SET status = 'published', published_at = NOW()
            WHERE id = ? AND status = 'draft'
        ");
        $stmt->bind_param('i', $postId);
        $stmt->execute();
        
        if ($stmt->affected_rows === 0) {
            throw new Exception("ไม่พบโพสต์ หรือโพสต์ไม่ได้อยู่ใน draft");
        }
        $stmt->close();
        
        // 2. ลบ tags เดิม
        $stmt = $db->prepare("DELETE FROM post_tags WHERE post_id = ?");
        $stmt->bind_param('i', $postId);
        $stmt->execute();
        $stmt->close();
        
        // 3. เพิ่ม tags ใหม่
        if (!empty($tagIds)) {
            $stmt = $db->prepare("INSERT INTO post_tags (post_id, tag_id) VALUES (?, ?)");
            foreach ($tagIds as $tagId) {
                $stmt->bind_param('ii', $postId, $tagId);
                $stmt->execute();
            }
            $stmt->close();
        }
        
        // 4. บันทึก activity log
        $stmt = $db->prepare("
            INSERT INTO activity_logs (action, entity_type, entity_id, created_at)
            VALUES ('publish', 'post', ?, NOW())
        ");
        $stmt->bind_param('i', $postId);
        $stmt->execute();
        $stmt->close();
        
        $db->commit();
        return true;
        
    } catch (Exception $e) {
        $db->rollback();
        throw $e;
    }
}
?>
```

---

## 🛠️ Workshop: Blog CRUD System

```php
<?php
// BlogController.php

class BlogController
{
    private mysqli $db;
    private int $postsPerPage = 10;
    
    public function __construct(mysqli $db)
    {
        $this->db = $db;
    }
    
    // ======================================
    // LIST: แสดงรายการโพสต์
    // ======================================
    
    public function index(int $page = 1, array $filters = []): array
    {
        $offset = ($page - 1) * $this->postsPerPage;
        $params = ['published'];
        $types = 's';
        $whereClauses = ["p.status = ?"];
        
        if (!empty($filters['category'])) {
            $whereClauses[] = "c.slug = ?";
            $params[] = $filters['category'];
            $types .= 's';
        }
        
        if (!empty($filters['search'])) {
            $whereClauses[] = "(p.title LIKE ? OR p.content LIKE ?)";
            $searchTerm = '%' . $filters['search'] . '%';
            $params[] = $searchTerm;
            $params[] = $searchTerm;
            $types .= 'ss';
        }
        
        $where = implode(' AND ', $whereClauses);
        
        // นับจำนวนทั้งหมด
        $countStmt = $this->db->prepare("
            SELECT COUNT(*) AS total
            FROM posts p
            LEFT JOIN categories c ON p.category_id = c.id
            WHERE $where
        ");
        $countStmt->bind_param($types, ...$params);
        $countStmt->execute();
        $total = $countStmt->get_result()->fetch_assoc()['total'];
        $countStmt->close();
        
        // ดึงข้อมูล
        $params[] = $this->postsPerPage;
        $params[] = $offset;
        $types .= 'ii';
        
        $stmt = $this->db->prepare("
            SELECT 
                p.id, p.title, p.slug, p.excerpt, p.views, p.published_at,
                u.name AS author,
                c.name AS category, c.slug AS category_slug,
                (SELECT COUNT(*) FROM comments cm WHERE cm.post_id = p.id AND cm.status = 'approved') AS comments_count
            FROM posts p
            LEFT JOIN users u ON p.user_id = u.id
            LEFT JOIN categories c ON p.category_id = c.id
            WHERE $where
            ORDER BY p.published_at DESC
            LIMIT ? OFFSET ?
        ");
        
        $stmt->bind_param($types, ...$params);
        $stmt->execute();
        $posts = $stmt->get_result()->fetch_all(MYSQLI_ASSOC);
        $stmt->close();
        
        return [
            'posts'       => $posts,
            'total'       => $total,
            'page'        => $page,
            'per_page'    => $this->postsPerPage,
            'total_pages' => ceil($total / $this->postsPerPage),
        ];
    }
    
    // ======================================
    // SHOW: แสดงโพสต์เดียว
    // ======================================
    
    public function show(string $slug): ?array
    {
        $stmt = $this->db->prepare("
            SELECT 
                p.*,
                u.name AS author_name,
                u.username AS author_username,
                c.name AS category_name,
                c.slug AS category_slug
            FROM posts p
            LEFT JOIN users u ON p.user_id = u.id
            LEFT JOIN categories c ON p.category_id = c.id
            WHERE p.slug = ? AND p.status = 'published'
        ");
        
        $stmt->bind_param('s', $slug);
        $stmt->execute();
        $post = $stmt->get_result()->fetch_assoc();
        $stmt->close();
        
        if (!$post) return null;
        
        // เพิ่ม views
        $this->incrementViews($post['id']);
        
        // ดึง comments
        $post['comments'] = $this->getComments($post['id']);
        
        return $post;
    }
    
    // ======================================
    // CREATE: สร้างโพสต์ใหม่
    // ======================================
    
    public function create(array $data): array
    {
        $errors = $this->validatePost($data);
        if (!empty($errors)) {
            return ['success' => false, 'errors' => $errors];
        }
        
        $slug = $this->generateUniqueSlug($data['title']);
        $excerpt = $data['excerpt'] ?? mb_substr(strip_tags($data['content']), 0, 200);
        $status = $data['status'] ?? 'draft';
        $publishedAt = $status === 'published' ? date('Y-m-d H:i:s') : null;
        
        $stmt = $this->db->prepare("
            INSERT INTO posts (title, slug, content, excerpt, user_id, category_id, status, published_at)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?)
        ");
        
        $stmt->bind_param(
            'ssssiiss',
            $data['title'],
            $slug,
            $data['content'],
            $excerpt,
            $data['user_id'],
            $data['category_id'],
            $status,
            $publishedAt
        );
        
        $stmt->execute();
        $newId = $this->db->insert_id;
        $stmt->close();
        
        return ['success' => true, 'id' => $newId, 'slug' => $slug];
    }
    
    // ======================================
    // UPDATE: แก้ไขโพสต์
    // ======================================
    
    public function update(int $postId, array $data, int $userId): array
    {
        // ตรวจสอบสิทธิ์
        $post = $this->findById($postId);
        if (!$post) {
            return ['success' => false, 'message' => 'ไม่พบโพสต์'];
        }
        
        if ($post['user_id'] !== $userId) {
            return ['success' => false, 'message' => 'ไม่มีสิทธิ์แก้ไขโพสต์นี้'];
        }
        
        $errors = $this->validatePost($data);
        if (!empty($errors)) {
            return ['success' => false, 'errors' => $errors];
        }
        
        $publishedAt = $post['published_at'];
        if ($data['status'] === 'published' && $post['status'] !== 'published') {
            $publishedAt = date('Y-m-d H:i:s');
        }
        
        $stmt = $this->db->prepare("
            UPDATE posts 
            SET title = ?, content = ?, excerpt = ?, category_id = ?, status = ?, published_at = ?
            WHERE id = ?
        ");
        
        $excerpt = $data['excerpt'] ?? mb_substr(strip_tags($data['content']), 0, 200);
        
        $stmt->bind_param(
            'ssssssi',
            $data['title'],
            $data['content'],
            $excerpt,
            $data['category_id'],
            $data['status'],
            $publishedAt,
            $postId
        );
        
        $stmt->execute();
        $affected = $stmt->affected_rows;
        $stmt->close();
        
        return ['success' => true, 'affected' => $affected];
    }
    
    // ======================================
    // DELETE: ลบโพสต์
    // ======================================
    
    public function delete(int $postId, int $userId): array
    {
        $post = $this->findById($postId);
        if (!$post) {
            return ['success' => false, 'message' => 'ไม่พบโพสต์'];
        }
        
        if ($post['user_id'] !== $userId) {
            return ['success' => false, 'message' => 'ไม่มีสิทธิ์ลบโพสต์นี้'];
        }
        
        // Soft delete
        $stmt = $this->db->prepare("UPDATE posts SET status = 'archived' WHERE id = ?");
        $stmt->bind_param('i', $postId);
        $stmt->execute();
        $stmt->close();
        
        return ['success' => true];
    }
    
    // ======================================
    // COMMENTS
    // ======================================
    
    public function addComment(int $postId, array $data): array
    {
        $content = trim($data['content'] ?? '');
        if (empty($content) || strlen($content) < 3) {
            return ['success' => false, 'message' => 'กรุณากรอกความเห็น'];
        }
        
        $stmt = $this->db->prepare("
            INSERT INTO comments (post_id, user_id, author_name, author_email, content, status)
            VALUES (?, ?, ?, ?, ?, 'pending')
        ");
        
        $userId = $data['user_id'] ?? null;
        $authorName = $data['author_name'] ?? 'ไม่ระบุ';
        $authorEmail = $data['author_email'] ?? '';
        
        $stmt->bind_param('iisss', $postId, $userId, $authorName, $authorEmail, $content);
        $stmt->execute();
        $stmt->close();
        
        return ['success' => true, 'message' => 'ความเห็นของคุณรอการอนุมัติ'];
    }
    
    private function getComments(int $postId): array
    {
        $stmt = $this->db->prepare("
            SELECT c.*, u.name AS user_name, u.avatar
            FROM comments c
            LEFT JOIN users u ON c.user_id = u.id
            WHERE c.post_id = ? AND c.status = 'approved'
            ORDER BY c.created_at ASC
        ");
        $stmt->bind_param('i', $postId);
        $stmt->execute();
        $result = $stmt->get_result()->fetch_all(MYSQLI_ASSOC);
        $stmt->close();
        return $result;
    }
    
    private function incrementViews(int $postId): void
    {
        $stmt = $this->db->prepare("UPDATE posts SET views = views + 1 WHERE id = ?");
        $stmt->bind_param('i', $postId);
        $stmt->execute();
        $stmt->close();
    }
    
    private function findById(int $id): ?array
    {
        $stmt = $this->db->prepare("SELECT * FROM posts WHERE id = ?");
        $stmt->bind_param('i', $id);
        $stmt->execute();
        $result = $stmt->get_result()->fetch_assoc();
        $stmt->close();
        return $result;
    }
    
    private function generateUniqueSlug(string $title): string
    {
        $baseSlug = createSlug($title);
        $slug = $baseSlug;
        $i = 1;
        
        while (true) {
            $stmt = $this->db->prepare("SELECT id FROM posts WHERE slug = ?");
            $stmt->bind_param('s', $slug);
            $stmt->execute();
            $exists = $stmt->get_result()->fetch_assoc();
            $stmt->close();
            
            if (!$exists) break;
            $slug = $baseSlug . '-' . $i++;
        }
        
        return $slug;
    }
    
    private function validatePost(array $data): array
    {
        $errors = [];
        
        if (empty($data['title']) || strlen(trim($data['title'])) < 3) {
            $errors['title'] = 'หัวข้อต้องมีอย่างน้อย 3 ตัวอักษร';
        }
        
        if (empty($data['content']) || strlen(strip_tags($data['content'])) < 10) {
            $errors['content'] = 'เนื้อหาต้องมีอย่างน้อย 10 ตัวอักษร';
        }
        
        return $errors;
    }
}

function createSlug(string $text): string
{
    $text = strtolower($text);
    $text = preg_replace('/[\s\-]+/', '-', $text);
    $text = preg_replace('/[^a-z0-9\-]/', '', $text);
    return trim($text, '-') ?: 'post-' . time();
}
?>
```

---

## 📝 Quiz

### คำถาม

**ข้อ 1:** Prepared Statement ป้องกันอะไร?
- A. XSS Attack
- B. SQL Injection
- C. CSRF Attack
- D. Brute Force

**ข้อ 2:** `$mysqli->insert_id` คืออะไร?
- A. จำนวนแถวที่ถูก insert
- B. ID ของแถวที่เพิ่งถูก insert
- C. ID ของ query ล่าสุด
- D. จำนวน queries ที่ทำ

**ข้อ 3:** `$db->begin_transaction()`, `$db->commit()`, `$db->rollback()` ใช้ทำอะไร?
- A. จัดการ Cache
- B. จัดการ Transaction
- C. จัดการ Connection Pool
- D. จัดการ Error Handling

**ข้อ 4:** `FOR UPDATE` ใน SELECT ใช้ทำอะไร?
- A. อัปเดตข้อมูลทันที
- B. Lock แถวที่ select ไว้ไม่ให้ process อื่น update ระหว่าง transaction
- C. เพิ่มความเร็วการ select
- D. บังคับให้ใช้ index

**ข้อ 5:** Soft Delete แตกต่างจาก Hard Delete อย่างไร?
- A. Soft Delete เร็วกว่า
- B. Soft Delete เปลี่ยน status แทนที่จะลบ ทำให้ recover ได้
- C. Soft Delete ใช้ Transaction
- D. Soft Delete ใช้ Prepared Statement

### เฉลย

| ข้อ | คำตอบ | อธิบาย |
|-----|-------|--------|
| 1 | B | Prepared Statements แยก SQL code ออกจาก data ทำให้ attacker ไม่สามารถ inject SQL code ได้ |
| 2 | B | `insert_id` คือ AUTO_INCREMENT ID ของแถวที่เพิ่งถูก insert ครั้งล่าสุด |
| 3 | B | Transaction ทำให้ queries หลายตัวสำเร็จทั้งหมดหรือ rollback ทั้งหมด |
| 4 | B | `FOR UPDATE` lock rows ที่ถูก select ไว้ใน transaction ป้องกัน race condition |
| 5 | B | Soft Delete เก็บข้อมูลไว้โดยเปลี่ยน status ทำให้สามารถ restore ได้ |

---

## ➡️ Part ถัดไป

**[Part 13: PHP PDO - PHP Data Objects](part-013-php-pdo.md)**
