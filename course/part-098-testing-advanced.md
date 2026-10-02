# Part 98: Testing Advanced ใน PHP

## บทนำ

Advanced Testing ครอบคลุมเทคนิคที่นอกเหนือจาก Unit Testing ธรรมดา เพื่อให้ Software มีความน่าเชื่อถือสูงสุด

---

## TDD (Test-Driven Development)

TDD เป็นกระบวนการ: **Red → Green → Refactor**

1. **Red**: เขียน Test ที่ Fail ก่อน
2. **Green**: เขียน Code ให้ Test ผ่าน
3. **Refactor**: ปรับปรุง Code โดยไม่ให้ Test Fail

### ตัวอย่าง TDD: Shopping Cart

```php
<?php

// Step 1: RED - เขียน Test ก่อน (ยังไม่มี CartItem class)
namespace Tests\Unit;

use App\Cart\{Cart, CartItem, Money};
use PHPUnit\Framework\TestCase;

class CartTest extends TestCase
{
    private Cart $cart;
    
    protected function setUp(): void
    {
        $this->cart = new Cart();
    }
    
    // ทดสอบ: Cart ว่างตั้งแต่ต้น
    public function test_cart_starts_empty(): void
    {
        $this->assertTrue($this->cart->isEmpty());
        $this->assertEquals(0, $this->cart->count());
        $this->assertEquals(Money::zero('THB'), $this->cart->total());
    }
    
    // ทดสอบ: เพิ่มสินค้า
    public function test_can_add_item(): void
    {
        $item = new CartItem(
            productId: 'PROD-1',
            name: 'Test Product',
            price: Money::of(100, 'THB'),
            quantity: 1
        );
        
        $this->cart->add($item);
        
        $this->assertFalse($this->cart->isEmpty());
        $this->assertEquals(1, $this->cart->count());
    }
    
    // ทดสอบ: คำนวณยอดรวม
    public function test_calculates_total_correctly(): void
    {
        $this->cart->add(new CartItem('P1', 'A', Money::of(100, 'THB'), 2));
        $this->cart->add(new CartItem('P2', 'B', Money::of(50, 'THB'), 3));
        
        // 100*2 + 50*3 = 200 + 150 = 350
        $this->assertEquals(Money::of(350, 'THB'), $this->cart->total());
    }
    
    // ทดสอบ: เพิ่มสินค้าที่มีอยู่แล้ว ควรเพิ่ม Quantity
    public function test_adding_same_product_increases_quantity(): void
    {
        $this->cart->add(new CartItem('P1', 'Product', Money::of(100, 'THB'), 2));
        $this->cart->add(new CartItem('P1', 'Product', Money::of(100, 'THB'), 3));
        
        $this->assertEquals(1, $this->cart->count()); // ยังมีแค่ 1 item
        $item = $this->cart->find('P1');
        $this->assertEquals(5, $item->getQuantity()); // แต่ quantity = 5
    }
    
    // ทดสอบ: ลบสินค้า
    public function test_can_remove_item(): void
    {
        $this->cart->add(new CartItem('P1', 'Product', Money::of(100, 'THB'), 1));
        $this->cart->remove('P1');
        
        $this->assertTrue($this->cart->isEmpty());
    }
    
    // ทดสอบ: ลบสินค้าที่ไม่มี ควร throw
    public function test_throws_when_removing_nonexistent_item(): void
    {
        $this->expectException(\InvalidArgumentException::class);
        $this->expectExceptionMessage("Product P999 not found in cart");
        
        $this->cart->remove('P999');
    }
    
    // ทดสอบ: Apply Discount
    public function test_applies_percentage_discount(): void
    {
        $this->cart->add(new CartItem('P1', 'Product', Money::of(1000, 'THB'), 1));
        $this->cart->applyDiscount(new PercentageDiscount(10));
        
        $this->assertEquals(Money::of(900, 'THB'), $this->cart->total());
    }
}

// Step 2: GREEN - เขียน Code ให้ Test ผ่าน
class Cart
{
    private array $items = [];
    private ?Discount $discount = null;
    
    public function isEmpty(): bool
    {
        return empty($this->items);
    }
    
    public function count(): int
    {
        return count($this->items);
    }
    
    public function add(CartItem $item): void
    {
        $productId = $item->getProductId();
        
        if (isset($this->items[$productId])) {
            $this->items[$productId] = $this->items[$productId]->withAddedQuantity(
                $item->getQuantity()
            );
        } else {
            $this->items[$productId] = $item;
        }
    }
    
    public function remove(string $productId): void
    {
        if (!isset($this->items[$productId])) {
            throw new \InvalidArgumentException("Product {$productId} not found in cart");
        }
        
        unset($this->items[$productId]);
    }
    
    public function find(string $productId): ?CartItem
    {
        return $this->items[$productId] ?? null;
    }
    
    public function total(): Money
    {
        $subtotal = array_reduce(
            $this->items,
            fn(Money $carry, CartItem $item) => $carry->add($item->getTotal()),
            Money::zero('THB')
        );
        
        if ($this->discount !== null) {
            $subtotal = $this->discount->apply($subtotal);
        }
        
        return $subtotal;
    }
    
    public function applyDiscount(Discount $discount): void
    {
        $this->discount = $discount;
    }
}

// Step 3: REFACTOR - ปรับปรุง Code
// ตอนนี้ Tests pass แล้ว เราสามารถ Refactor ได้อย่างมั่นใจ
```

---

## BDD (Behavior-Driven Development) ด้วย Behat

```bash
composer require --dev behat/behat
vendor/bin/behat --init
```

### Feature Files (Gherkin)

```gherkin
# features/shopping_cart.feature

Feature: Shopping Cart
  As a customer
  I want to manage my shopping cart
  So that I can purchase multiple products at once

  Background:
    Given I am a logged-in customer
    And the following products exist:
      | name          | price | stock |
      | Laptop        | 29900 | 10    |
      | Mouse         | 590   | 50    |
      | USB Hub       | 390   | 30    |

  Scenario: Add a single product to cart
    When I add 1 "Laptop" to my cart
    Then my cart should contain 1 item
    And the cart total should be 29900 THB

  Scenario: Add multiple products to cart
    When I add 1 "Laptop" to my cart
    And I add 2 "Mouse" to my cart
    Then my cart should contain 2 items
    And the cart total should be 31080 THB

  Scenario: Remove product from cart
    Given I have 1 "Laptop" in my cart
    When I remove "Laptop" from my cart
    Then my cart should be empty

  Scenario: Apply coupon discount
    Given I have 1 "Laptop" in my cart
    When I apply coupon "SAVE10"
    Then the cart total should be 26910 THB
    And I should see a discount of 2990 THB

  Scenario Outline: Quantity validation
    When I try to add <quantity> "Mouse" to my cart
    Then I should see an error "<error>"

    Examples:
      | quantity | error                          |
      | 0        | Quantity must be at least 1    |
      | -1       | Quantity must be at least 1    |
      | 100      | Insufficient stock             |

  Scenario: Out of stock product
    Given product "Laptop" has 0 stock
    When I try to add 1 "Laptop" to my cart
    Then I should see "Product is out of stock"
```

### Step Definitions

```php
<?php

namespace App\Tests\Behat;

use Behat\Behat\Context\Context;
use Behat\Behat\Tester\Exception\PendingException;
use Behat\Gherkin\Node\TableNode;
use PHPUnit\Framework\Assert;

class CartContext implements Context
{
    private array $products = [];
    private Cart $cart;
    private ?string $lastError = null;
    
    public function __construct()
    {
        $this->cart = new Cart();
    }
    
    /**
     * @Given I am a logged-in customer
     */
    public function iAmALoggedInCustomer(): void
    {
        // Setup authenticated user context
    }
    
    /**
     * @Given the following products exist:
     */
    public function theFollowingProductsExist(TableNode $table): void
    {
        foreach ($table->getHash() as $row) {
            $this->products[$row['name']] = new Product(
                id: uniqid(),
                name: $row['name'],
                price: (float)$row['price'],
                stock: (int)$row['stock']
            );
        }
    }
    
    /**
     * @When I add :quantity :productName to my cart
     */
    public function iAddProductToMyCart(int $quantity, string $productName): void
    {
        $product = $this->products[$productName]
            ?? throw new \InvalidArgumentException("Product not found: {$productName}");
        
        try {
            $this->cart->add(new CartItem(
                productId: $product->id,
                name: $product->name,
                price: Money::of($product->price, 'THB'),
                quantity: $quantity
            ));
        } catch (\Exception $e) {
            $this->lastError = $e->getMessage();
        }
    }
    
    /**
     * @When I try to add :quantity :productName to my cart
     */
    public function iTryToAddProductToMyCart(int $quantity, string $productName): void
    {
        $this->iAddProductToMyCart($quantity, $productName);
    }
    
    /**
     * @Then my cart should contain :count item(s)
     */
    public function myCartShouldContainItems(int $count): void
    {
        Assert::assertEquals($count, $this->cart->count());
    }
    
    /**
     * @Then the cart total should be :total THB
     */
    public function theCartTotalShouldBe(float $total): void
    {
        Assert::assertEquals(
            Money::of($total, 'THB'),
            $this->cart->total()
        );
    }
    
    /**
     * @Then my cart should be empty
     */
    public function myCartShouldBeEmpty(): void
    {
        Assert::assertTrue($this->cart->isEmpty());
    }
    
    /**
     * @When I apply coupon :code
     */
    public function iApplyCoupon(string $code): void
    {
        $this->cart->applyCoupon($code);
    }
    
    /**
     * @Then I should see an error :message
     */
    public function iShouldSeeAnError(string $message): void
    {
        Assert::assertEquals($message, $this->lastError);
    }
    
    /**
     * @Then I should see :message
     */
    public function iShouldSee(string $message): void
    {
        Assert::assertStringContainsString($message, $this->lastError ?? '');
    }
}
```

---

## Mutation Testing

Mutation Testing ทดสอบว่า Tests ของเรา "ดีแค่ไหน" โดยแก้ Code เล็กน้อย (Mutations) แล้วดูว่า Tests ยัง Fail อยู่ไหม

```bash
composer require --dev infection/infection
vendor/bin/infection --min-msi=70 --min-covered-msi=80
```

### Mutation Types ที่ Infection ใช้

```php
<?php

// Original Code
function isEligibleForDiscount(int $age, float $totalAmount): bool
{
    return $age >= 60 || $totalAmount >= 1000;
}

// Mutations ที่ Infection จะลอง:
// 1. เปลี่ยน >= เป็น >
function isEligibleForDiscount_Mutation1(int $age, float $totalAmount): bool
{
    return $age > 60 || $totalAmount > 1000; // Mutated
}

// 2. เปลี่ยน || เป็น &&
function isEligibleForDiscount_Mutation2(int $age, float $totalAmount): bool
{
    return $age >= 60 && $totalAmount >= 1000; // Mutated
}

// 3. ลบ Return
function isEligibleForDiscount_Mutation3(int $age, float $totalAmount): bool
{
    return false; // Mutated
}

// Tests ที่ดีต้อง Catch Mutations เหล่านี้ได้ทั้งหมด

class DiscountEligibilityTest extends TestCase
{
    // Test นี้จับ Mutation 1 (>= vs >)
    public function test_exactly_60_is_eligible(): void
    {
        $this->assertTrue(isEligibleForDiscount(60, 500));
    }
    
    // Test นี้จับ Mutation 2 (|| vs &&)
    public function test_high_amount_is_eligible_without_senior_age(): void
    {
        $this->assertTrue(isEligibleForDiscount(30, 1000));
    }
    
    // Test นี้จับ Mutation 3 (always false)
    public function test_eligible_when_both_conditions_met(): void
    {
        $this->assertTrue(isEligibleForDiscount(65, 2000));
    }
    
    // Test สำหรับ false case
    public function test_not_eligible_when_young_and_low_amount(): void
    {
        $this->assertFalse(isEligibleForDiscount(25, 500));
    }
}
```

---

## Contract Testing

Contract Testing ตรวจสอบว่า API ที่ Services ใช้ร่วมกันยังคง Contract เดิมอยู่

```bash
composer require --dev pact-foundation/pact-php
```

```php
<?php

use PhpPact\Consumer\InteractionBuilder;
use PhpPact\Consumer\Model\ConsumerRequest;
use PhpPact\Consumer\Model\ProviderResponse;
use PhpPact\Consumer\MockServer;
use PhpPact\Consumer\MockServerEnvConfig;

// Consumer Test (Order Service ที่เรียก Product Service)
class OrderServiceConsumerTest extends TestCase
{
    private MockServer $mockServer;
    private InteractionBuilder $builder;
    
    protected function setUp(): void
    {
        $config = new MockServerEnvConfig();
        $this->mockServer = new MockServer($config);
        $this->mockServer->start();
        $this->builder = new InteractionBuilder($config);
    }
    
    public function test_gets_product_details(): void
    {
        // Define expected interaction
        $request = new ConsumerRequest();
        $request
            ->setMethod('GET')
            ->setPath('/api/products/1')
            ->addHeader('Accept', 'application/json');
        
        $response = new ProviderResponse();
        $response
            ->setStatus(200)
            ->addHeader('Content-Type', 'application/json')
            ->setBody([
                'id' => 1,
                'name' => 'Test Product',
                'price' => 100.0,
                'stock' => 50,
            ]);
        
        $this->builder
            ->given('Product 1 exists')
            ->uponReceiving('a request for product 1')
            ->with($request)
            ->willRespondWith($response);
        
        // Run actual code against mock server
        $client = new ProductServiceClient($this->mockServer->getBaseUri());
        $product = $client->findProduct(1);
        
        $this->assertEquals(1, $product['id']);
        $this->assertEquals('Test Product', $product['name']);
        $this->assertEquals(100.0, $product['price']);
        
        // Verify interactions
        $this->mockServer->verify();
    }
    
    public function test_returns_null_for_nonexistent_product(): void
    {
        $request = new ConsumerRequest();
        $request->setMethod('GET')->setPath('/api/products/999');
        
        $response = new ProviderResponse();
        $response->setStatus(404)->setBody(['message' => 'Not found']);
        
        $this->builder
            ->given('Product 999 does not exist')
            ->uponReceiving('a request for nonexistent product')
            ->with($request)
            ->willRespondWith($response);
        
        $client = new ProductServiceClient($this->mockServer->getBaseUri());
        $product = $client->findProduct(999);
        
        $this->assertNull($product);
    }
    
    protected function tearDown(): void
    {
        $this->mockServer->stop();
    }
}
```

---

## Performance Testing

```bash
composer require --dev phpbench/phpbench
```

```php
<?php

use PhpBench\Attributes as Bench;

#[Bench\Revs(1000)]
#[Bench\Iterations(5)]
#[Bench\Warmup(2)]
class SortingBench
{
    private array $data;
    
    public function setUp(): void
    {
        $this->data = range(1, 1000);
        shuffle($this->data);
    }
    
    #[Bench\Subject]
    #[Bench\BeforeClassMethods(['setUp'])]
    public function benchBubbleSort(): void
    {
        $data = $this->data;
        $n = count($data);
        for ($i = 0; $i < $n - 1; $i++) {
            for ($j = 0; $j < $n - $i - 1; $j++) {
                if ($data[$j] > $data[$j + 1]) {
                    [$data[$j], $data[$j + 1]] = [$data[$j + 1], $data[$j]];
                }
            }
        }
    }
    
    #[Bench\Subject]
    #[Bench\BeforeClassMethods(['setUp'])]
    public function benchQuickSort(): void
    {
        $data = $this->data;
        $this->quickSort($data, 0, count($data) - 1);
    }
    
    #[Bench\Subject]
    #[Bench\BeforeClassMethods(['setUp'])]
    public function benchPhpNativeSort(): void
    {
        $data = $this->data;
        sort($data);
    }
    
    private function quickSort(array &$arr, int $low, int $high): void
    {
        if ($low < $high) {
            $pivot = $this->partition($arr, $low, $high);
            $this->quickSort($arr, $low, $pivot - 1);
            $this->quickSort($arr, $pivot + 1, $high);
        }
    }
    
    private function partition(array &$arr, int $low, int $high): int
    {
        $pivot = $arr[$high];
        $i = $low - 1;
        
        for ($j = $low; $j < $high; $j++) {
            if ($arr[$j] <= $pivot) {
                $i++;
                [$arr[$i], $arr[$j]] = [$arr[$j], $arr[$i]];
            }
        }
        
        [$arr[$i + 1], $arr[$high]] = [$arr[$high], $arr[$i + 1]];
        return $i + 1;
    }
}

// รัน: vendor/bin/phpbench run --report=default
```

---

## Workshop: Full Test Suite สำหรับ API

```php
<?php

namespace Tests;

use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;
use App\Models\{User, Product, Order, Category};

// Feature Test สำหรับ Order API
class OrderApiTest extends TestCase
{
    use RefreshDatabase;
    
    private User $user;
    private User $admin;
    
    protected function setUp(): void
    {
        parent::setUp();
        
        $this->user = User::factory()->create(['role' => 'customer']);
        $this->admin = User::factory()->create(['role' => 'admin']);
    }
    
    // ======= Place Order =======
    
    public function test_authenticated_user_can_place_order(): void
    {
        $product = Product::factory()->create(['price' => 100, 'stock' => 10]);
        
        $response = $this->actingAs($this->user)
            ->postJson('/api/orders', [
                'items' => [
                    ['product_id' => $product->id, 'quantity' => 2]
                ],
                'shipping_address' => [
                    'street' => '123 Test Street',
                    'city' => 'Bangkok',
                    'postal_code' => '10110',
                    'country' => 'TH',
                ],
                'payment_method' => 'credit_card',
            ]);
        
        $response->assertStatus(201)
            ->assertJsonStructure([
                'data' => [
                    'id',
                    'status',
                    'items',
                    'total',
                    'created_at',
                ]
            ])
            ->assertJsonPath('data.status', 'pending')
            ->assertJsonPath('data.total', 200.0);
        
        $this->assertDatabaseHas('orders', [
            'user_id' => $this->user->id,
            'total' => 200.0,
        ]);
        
        // ตรวจสอบ Stock ลดลง
        $this->assertDatabaseHas('products', [
            'id' => $product->id,
            'stock' => 8, // 10 - 2
        ]);
    }
    
    public function test_guest_cannot_place_order(): void
    {
        $response = $this->postJson('/api/orders', []);
        
        $response->assertStatus(401);
    }
    
    public function test_validates_order_items(): void
    {
        $response = $this->actingAs($this->user)
            ->postJson('/api/orders', [
                'items' => [], // ต้องมี items
            ]);
        
        $response->assertStatus(422)
            ->assertJsonValidationErrors(['items']);
    }
    
    public function test_cannot_order_out_of_stock_product(): void
    {
        $product = Product::factory()->create(['stock' => 0]);
        
        $response = $this->actingAs($this->user)
            ->postJson('/api/orders', [
                'items' => [
                    ['product_id' => $product->id, 'quantity' => 1]
                ],
            ]);
        
        $response->assertStatus(422)
            ->assertJsonPath('message', 'Insufficient stock');
    }
    
    // ======= Get Orders =======
    
    public function test_user_can_only_see_their_orders(): void
    {
        Order::factory()->count(3)->create(['user_id' => $this->user->id]);
        Order::factory()->count(5)->create(['user_id' => $this->admin->id]);
        
        $response = $this->actingAs($this->user)
            ->getJson('/api/orders');
        
        $response->assertStatus(200);
        $this->assertCount(3, $response->json('data'));
    }
    
    public function test_admin_can_see_all_orders(): void
    {
        Order::factory()->count(3)->create(['user_id' => $this->user->id]);
        Order::factory()->count(5)->create(['user_id' => $this->admin->id]);
        
        $response = $this->actingAs($this->admin)
            ->getJson('/api/admin/orders');
        
        $response->assertStatus(200);
        $this->assertEquals(8, $response->json('meta.total'));
    }
    
    // ======= Cancel Order =======
    
    public function test_user_can_cancel_pending_order(): void
    {
        $order = Order::factory()->create([
            'user_id' => $this->user->id,
            'status' => 'pending',
        ]);
        
        $response = $this->actingAs($this->user)
            ->patchJson("/api/orders/{$order->id}/cancel");
        
        $response->assertStatus(200)
            ->assertJsonPath('data.status', 'cancelled');
    }
    
    public function test_cannot_cancel_shipped_order(): void
    {
        $order = Order::factory()->create([
            'user_id' => $this->user->id,
            'status' => 'shipped',
        ]);
        
        $response = $this->actingAs($this->user)
            ->patchJson("/api/orders/{$order->id}/cancel");
        
        $response->assertStatus(422)
            ->assertJsonPath('message', 'Cannot cancel a shipped order');
    }
    
    public function test_user_cannot_cancel_other_users_order(): void
    {
        $otherUser = User::factory()->create();
        $order = Order::factory()->create([
            'user_id' => $otherUser->id,
            'status' => 'pending',
        ]);
        
        $response = $this->actingAs($this->user)
            ->patchJson("/api/orders/{$order->id}/cancel");
        
        $response->assertStatus(403);
    }
}

// Test สำหรับ Jobs
class ProcessOrderJobTest extends TestCase
{
    use RefreshDatabase;
    
    public function test_processes_order_successfully(): void
    {
        Queue::fake();
        
        $order = Order::factory()->create(['status' => 'pending']);
        
        ProcessOrderJob::dispatch($order->id, [
            'method' => 'credit_card',
            'token' => 'tok_visa',
        ]);
        
        Queue::assertPushed(ProcessOrderJob::class, function ($job) use ($order) {
            return $job->orderId === $order->id;
        });
    }
    
    public function test_sends_confirmation_email_after_processing(): void
    {
        Mail::fake();
        
        $order = Order::factory()->create(['status' => 'pending']);
        
        // Run job directly
        (new ProcessOrderJob($order->id, ['method' => 'credit_card']))->handle(
            app(OrderService::class),
            app(PaymentService::class)
        );
        
        Mail::assertSent(OrderConfirmationMail::class, function ($mail) use ($order) {
            return $mail->order->id === $order->id;
        });
    }
}
```

---

## สรุป

| Testing Type | เครื่องมือ | วัตถุประสงค์ |
|------------|----------|------------|
| Unit Test | PHPUnit, Pest | Test Functions/Methods |
| Feature Test | Laravel TestCase | Test HTTP Endpoints |
| TDD | PHPUnit + Process | Guide Development |
| BDD | Behat | Business Requirements |
| Mutation Testing | Infection | Test Quality |
| Contract Testing | Pact | API Contracts |
| Performance | PHPBench | Speed/Memory |
| E2E | Laravel Dusk | Full User Flow |

---

*"Test code is production code" - Tests ที่ดีเป็น Investment ที่คุ้มค่าที่สุด*
