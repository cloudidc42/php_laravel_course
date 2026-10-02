# Part 95: GraphQL ใน PHP

## บทนำ

GraphQL คือ Query Language สำหรับ API ที่พัฒนาโดย Facebook ช่วยให้ Client สามารถ Request ข้อมูลเฉพาะที่ต้องการได้ แทนที่จะรับข้อมูลทั้งหมดเหมือน REST

**ข้อดี GraphQL เทียบกับ REST**:
- ได้ข้อมูลตรงที่ต้องการ (No over-fetching / under-fetching)
- Single Endpoint
- Strongly Typed Schema
- Real-time ด้วย Subscriptions
- Self-documenting

---

## การติดตั้ง Lighthouse

Lighthouse เป็น GraphQL Server Library ที่ดีที่สุดสำหรับ Laravel

```bash
composer require nuwave/lighthouse
php artisan vendor:publish --tag=lighthouse-schema
php artisan vendor:publish --tag=lighthouse-config
```

---

## Schema Definition

### Basic Schema

```graphql
# graphql/schema.graphql

type Query {
  # ดึง User เดี่ยว
  user(id: ID! @eq): User @find
  
  # ดึง Users ทั้งหมด
  users(
    name: String @where(operator: "LIKE", value: "%{name}%")
    email: String @eq
    first: Int @first
  ): [User!]! @all @paginate(type: PAGINATOR)
  
  # ดึง Products พร้อม Pagination
  products(
    first: Int = 20
    page: Int
    category_id: ID @eq
    min_price: Float @where(operator: ">=", key: "price")
    max_price: Float @where(operator: "<=", key: "price")
    search: String @search
  ): ProductPaginator @paginate(type: PAGINATOR)
  
  # Custom Query
  featuredProducts: [Product!]! @field(resolver: "App\\GraphQL\\Queries\\FeaturedProductsQuery")
  
  # Current User
  me: User @auth
}

type Mutation {
  # User
  register(input: RegisterInput! @spread): AuthPayload! 
    @field(resolver: "App\\GraphQL\\Mutations\\RegisterMutation")
  
  login(email: String!, password: String!): AuthPayload!
    @field(resolver: "App\\GraphQL\\Mutations\\LoginMutation")
  
  logout: Boolean! @field(resolver: "App\\GraphQL\\Mutations\\LogoutMutation") @auth
  
  # User Profile
  updateProfile(input: UpdateProfileInput! @spread): User!
    @update @auth
  
  # Products (Admin only)
  createProduct(input: CreateProductInput! @spread): Product!
    @create @can(ability: "create", model: "App\\Models\\Product")
  
  updateProduct(id: ID!, input: UpdateProductInput! @spread): Product!
    @update @can(ability: "update", find: "id", model: "App\\Models\\Product")
  
  deleteProduct(id: ID!): Product
    @delete @can(ability: "delete", find: "id", model: "App\\Models\\Product")
  
  # Orders
  placeOrder(input: PlaceOrderInput! @spread): Order!
    @field(resolver: "App\\GraphQL\\Mutations\\PlaceOrderMutation") @auth
}

type Subscription {
  orderStatusUpdated(orderId: ID!): Order
    @subscription(class: "App\\GraphQL\\Subscriptions\\OrderStatusSubscription")
  
  newMessage(conversationId: ID!): Message
    @subscription(class: "App\\GraphQL\\Subscriptions\\NewMessageSubscription")
}
```

### Type Definitions

```graphql
# Types

type User {
  id: ID!
  name: String!
  email: String!
  role: UserRole!
  profile: UserProfile
  orders(first: Int = 10, page: Int): OrderPaginator @hasMany @paginate(type: PAGINATOR)
  reviews: [Review!]! @hasMany
  created_at: DateTime!
  updated_at: DateTime!
}

type UserProfile {
  id: ID!
  bio: String
  avatar: String
  phone: String
  address: Address
  user: User! @belongsTo
}

type Product {
  id: ID!
  name: String!
  description: String!
  price: Float!
  stock: Int!
  category: Category @belongsTo
  images: [ProductImage!]! @hasMany
  reviews: [Review!]! @hasMany
  avg_rating: Float @method(name: "averageRating")
  review_count: Int @count(relation: "reviews")
  created_at: DateTime!
  is_available: Boolean! @method(name: "isAvailable")
}

type Order {
  id: ID!
  status: OrderStatus!
  items: [OrderItem!]! @hasMany
  subtotal: Float!
  tax: Float!
  total: Float!
  shipping_address: Address!
  user: User! @belongsTo
  created_at: DateTime!
  updated_at: DateTime!
}

type OrderItem {
  id: ID!
  product: Product! @belongsTo
  quantity: Int!
  unit_price: Float!
  total: Float!
}

type Category {
  id: ID!
  name: String!
  slug: String!
  parent: Category @belongsTo
  children: [Category!]! @hasMany
  products(first: Int = 20, page: Int): ProductPaginator @hasMany @paginate(type: PAGINATOR)
}

type Review {
  id: ID!
  rating: Int!
  comment: String
  user: User! @belongsTo
  product: Product! @belongsTo
  created_at: DateTime!
}

type AuthPayload {
  access_token: String!
  token_type: String!
  expires_in: Int!
  user: User!
}

type Address {
  street: String!
  city: String!
  state: String
  postal_code: String!
  country: String!
}

# Enums
enum UserRole {
  ADMIN
  CUSTOMER
  MODERATOR
}

enum OrderStatus {
  PENDING
  CONFIRMED
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
  REFUNDED
}

# Inputs
input RegisterInput {
  name: String! @rules(apply: ["required", "min:2", "max:255"])
  email: String! @rules(apply: ["required", "email", "unique:users,email"])
  password: String! @rules(apply: ["required", "min:8", "confirmed"])
  password_confirmation: String!
}

input UpdateProfileInput {
  name: String @rules(apply: ["min:2", "max:255"])
  phone: String @rules(apply: ["nullable", "regex:/^[0-9+\\-\\s()]+$/"])
  bio: String @rules(apply: ["nullable", "max:500"])
}

input PlaceOrderInput {
  items: [OrderItemInput!]! @rulesForArray(apply: ["min:1"])
  shipping_address: AddressInput!
  payment_method: String! @rules(apply: ["in:credit_card,paypal,bank_transfer"])
}

input OrderItemInput {
  product_id: ID!
  quantity: Int! @rules(apply: ["min:1", "max:100"])
}

input AddressInput {
  street: String!
  city: String!
  postal_code: String!
  country: String! @rules(apply: ["size:2"])
}

input CreateProductInput {
  name: String! @rules(apply: ["required", "min:2", "max:255"])
  description: String! @rules(apply: ["required", "min:10"])
  price: Float! @rules(apply: ["min:0"])
  stock: Int! @rules(apply: ["min:0"])
  category_id: ID!
}

input UpdateProductInput {
  name: String @rules(apply: ["min:2", "max:255"])
  description: String @rules(apply: ["min:10"])
  price: Float @rules(apply: ["min:0"])
  stock: Int @rules(apply: ["min:0"])
}

# Pagination
type ProductPaginator {
  data: [Product!]!
  paginatorInfo: PaginatorInfo!
}

type OrderPaginator {
  data: [Order!]!
  paginatorInfo: PaginatorInfo!
}

type PaginatorInfo {
  count: Int!
  currentPage: Int!
  firstItem: Int
  hasMorePages: Boolean!
  lastItem: Int
  lastPage: Int!
  perPage: Int!
  total: Int!
}
```

---

## Resolvers

### Query Resolver

```php
<?php

namespace App\GraphQL\Queries;

use App\Models\Product;

class FeaturedProductsQuery
{
    public function __invoke($rootValue, array $args): \Illuminate\Database\Eloquent\Collection
    {
        return Product::where('is_featured', true)
            ->where('stock', '>', 0)
            ->where('status', 'active')
            ->with(['images', 'category'])
            ->orderByDesc('updated_at')
            ->limit(10)
            ->get();
    }
}

class ProductSearchQuery
{
    public function __invoke($rootValue, array $args): array
    {
        $query = Product::query()
            ->with(['category', 'images'])
            ->where('status', 'active');
        
        if (!empty($args['search'])) {
            $query->where(function ($q) use ($args) {
                $q->where('name', 'LIKE', "%{$args['search']}%")
                  ->orWhere('description', 'LIKE', "%{$args['search']}%");
            });
        }
        
        if (!empty($args['category_id'])) {
            $query->where('category_id', $args['category_id']);
        }
        
        if (!empty($args['min_price'])) {
            $query->where('price', '>=', $args['min_price']);
        }
        
        if (!empty($args['max_price'])) {
            $query->where('price', '<=', $args['max_price']);
        }
        
        $sortField = $args['sort_by'] ?? 'created_at';
        $sortDir = $args['sort_dir'] ?? 'desc';
        
        return $query->orderBy($sortField, $sortDir)->get()->toArray();
    }
}
```

### Mutation Resolvers

```php
<?php

namespace App\GraphQL\Mutations;

use App\Models\User;
use Illuminate\Support\Facades\{Auth, Hash};
use Nuwave\Lighthouse\Support\Contracts\GraphQLContext;

class RegisterMutation
{
    public function __invoke($rootValue, array $args, GraphQLContext $context): array
    {
        $user = User::create([
            'name' => $args['name'],
            'email' => $args['email'],
            'password' => Hash::make($args['password']),
        ]);
        
        $token = $user->createToken('graphql-token');
        
        return [
            'access_token' => $token->plainTextToken,
            'token_type' => 'Bearer',
            'expires_in' => 0, // Non-expiring
            'user' => $user,
        ];
    }
}

class LoginMutation
{
    public function __invoke($rootValue, array $args): array
    {
        $user = User::where('email', $args['email'])->first();
        
        if (!$user || !Hash::check($args['password'], $user->password)) {
            throw new \Nuwave\Lighthouse\Exceptions\AuthenticationException(
                "Invalid credentials"
            );
        }
        
        // Revoke old tokens
        $user->tokens()->delete();
        
        $token = $user->createToken('graphql-token');
        
        return [
            'access_token' => $token->plainTextToken,
            'token_type' => 'Bearer',
            'expires_in' => 0,
            'user' => $user,
        ];
    }
}

class PlaceOrderMutation
{
    public function __construct(
        private OrderService $orderService,
        private PaymentService $paymentService
    ) {}
    
    public function __invoke($rootValue, array $args, GraphQLContext $context): Order
    {
        $user = $context->user();
        
        return $this->orderService->placeOrder(
            userId: $user->id,
            items: $args['items'],
            shippingAddress: $args['shipping_address'],
            paymentMethod: $args['payment_method']
        );
    }
}
```

---

## Custom Directives

```php
<?php

namespace App\GraphQL\Directives;

use Nuwave\Lighthouse\Schema\Directives\BaseDirective;
use Nuwave\Lighthouse\Support\Contracts\{FieldMiddleware};
use Nuwave\Lighthouse\Execution\ResolveInfo;
use Nuwave\Lighthouse\Support\Contracts\GraphQLContext;

class RateLimitDirective extends BaseDirective implements FieldMiddleware
{
    public static function definition(): string
    {
        return /** @lang GraphQL */ <<<'GRAPHQL'
        """Limit the number of times this field can be called per minute."""
        directive @rateLimit(
            "Maximum number of calls per minute."
            limit: Int! = 60
        ) on FIELD_DEFINITION
        GRAPHQL;
    }
    
    public function handleField(FieldValue $fieldValue): void
    {
        $resolver = $fieldValue->getResolver();
        
        $fieldValue->setResolver(
            function ($root, array $args, GraphQLContext $context, ResolveInfo $info) use ($resolver) {
                $user = $context->user();
                $key = 'graphql_rate_limit:' . ($user?->id ?? $context->request()->ip())
                    . ':' . $info->fieldName;
                
                $limit = $this->directiveArgValue('limit', 60);
                $current = (int)\Cache::get($key, 0);
                
                if ($current >= $limit) {
                    throw new \Exception("Rate limit exceeded for field: {$info->fieldName}");
                }
                
                \Cache::put($key, $current + 1, 60);
                
                return $resolver($root, $args, $context, $info);
            }
        );
    }
}

// ใช้งาน Directive ใน Schema
// expensiveQuery: [Result!]! @rateLimit(limit: 5)
```

---

## Subscriptions

```php
<?php

namespace App\GraphQL\Subscriptions;

use App\Models\Order;
use Illuminate\Http\Request;
use Nuwave\Lighthouse\Schema\Types\GraphQLSubscription;
use Nuwave\Lighthouse\Subscriptions\Subscriber;

class OrderStatusSubscription extends GraphQLSubscription
{
    public function authorize(Subscriber $subscriber, Request $request): bool
    {
        $user = $request->user('sanctum');
        
        if (!$user) return false;
        
        // Check if user owns this order
        $orderId = $subscriber->args['orderId'];
        return Order::where('id', $orderId)
            ->where('user_id', $user->id)
            ->exists();
    }
    
    public function filter(Subscriber $subscriber, $root): bool
    {
        // Only send updates if order ID matches
        return $root->id === $subscriber->args['orderId'];
    }
}

// Broadcast Updates
class OrderController extends Controller
{
    public function updateStatus(Request $request, Order $order): JsonResponse
    {
        $order->update(['status' => $request->status]);
        
        // Broadcast to GraphQL subscribers
        \Broadcast::channel('order.' . $order->id, function ($user) use ($order) {
            return $user->id === $order->user_id;
        });
        
        // Trigger subscription
        app(\Nuwave\Lighthouse\Subscriptions\SubscriptionBroadcast::class)
            ->broadcast('orderStatusUpdated', $order);
        
        return response()->json(['success' => true]);
    }
}
```

---

## Workshop: สร้าง GraphQL API

### Client Queries ตัวอย่าง

```graphql
# Queries

# ดึง Products พร้อม Category
query GetProducts($first: Int, $page: Int, $search: String) {
  products(first: $first, page: $page) {
    data {
      id
      name
      price
      avg_rating
      review_count
      category {
        id
        name
      }
      images {
        url
        alt
      }
    }
    paginatorInfo {
      total
      hasMorePages
      currentPage
    }
  }
}

# ดึง User Profile พร้อม Recent Orders
query GetUserProfile {
  me {
    id
    name
    email
    profile {
      bio
      avatar
      phone
    }
    orders(first: 5) {
      data {
        id
        status
        total
        created_at
        items {
          quantity
          unit_price
          product {
            name
            images(limit: 1) {
              url
            }
          }
        }
      }
      paginatorInfo {
        total
      }
    }
  }
}

# Mutation - Place Order
mutation PlaceOrder($items: [OrderItemInput!]!, $address: AddressInput!, $payment: String!) {
  placeOrder(input: {
    items: $items,
    shipping_address: $address,
    payment_method: $payment
  }) {
    id
    status
    total
    items {
      product { name }
      quantity
      unit_price
    }
  }
}

# Subscription - Real-time Order Updates
subscription OnOrderUpdate($orderId: ID!) {
  orderStatusUpdated(orderId: $orderId) {
    id
    status
    updated_at
  }
}
```

### N+1 Prevention ด้วย DataLoader

```php
<?php

namespace App\GraphQL\Loaders;

use Illuminate\Support\Collection;
use Nuwave\Lighthouse\Execution\BatchLoader\BatchLoaderRegistry;

class CategoryLoader
{
    public static function load(array $keys): array
    {
        // โหลด Categories ทั้งหมดในครั้งเดียว
        $categories = Category::whereIn('id', $keys)->get()->keyBy('id');
        
        return array_map(
            fn($key) => $categories[$key] ?? null,
            $keys
        );
    }
}

// ใช้ใน Resolver
class ProductCategoryResolver
{
    public function __invoke(Product $product, array $args, GraphQLContext $context): ?Category
    {
        // Lighthouse จัดการ DataLoader อัตโนมัติผ่าน @belongsTo
        return $product->category;
    }
}
```

### Authentication & Authorization

```php
<?php

// config/lighthouse.php
'guard' => 'sanctum',

// Schema - ใช้ @auth directive
type Query {
  me: User @auth
  adminDashboard: Dashboard @auth @can(ability: "access-admin")
}

type Mutation {
  updateProfile(input: UpdateProfileInput! @spread): User! 
    @update 
    @auth 
    @inject(context: "user.id", name: "id")
}

// Policy
class ProductPolicy
{
    public function create(User $user): bool
    {
        return $user->role === 'admin';
    }
    
    public function update(User $user, Product $product): bool
    {
        return $user->role === 'admin';
    }
    
    public function delete(User $user, Product $product): bool
    {
        return $user->role === 'admin';
    }
}
```

---

## Testing GraphQL

```php
<?php

namespace Tests\Feature\GraphQL;

use Tests\TestCase;
use Illuminate\Foundation\Testing\RefreshDatabase;
use App\Models\{User, Product, Category};

class ProductQueryTest extends TestCase
{
    use RefreshDatabase;
    
    public function test_can_query_products(): void
    {
        Category::factory()->create(['id' => 1, 'name' => 'Electronics']);
        Product::factory()->count(5)->create(['category_id' => 1]);
        
        $response = $this->graphQL('
            query {
                products(first: 10) {
                    data {
                        id
                        name
                        price
                        category {
                            name
                        }
                    }
                    paginatorInfo {
                        total
                    }
                }
            }
        ');
        
        $response->assertJsonCount(5, 'data.products.data');
        $response->assertJson([
            'data' => [
                'products' => [
                    'paginatorInfo' => [
                        'total' => 5
                    ]
                ]
            ]
        ]);
    }
    
    public function test_can_register_user(): void
    {
        $response = $this->graphQL('
            mutation {
                register(input: {
                    name: "Test User"
                    email: "test@example.com"
                    password: "password123"
                    password_confirmation: "password123"
                }) {
                    access_token
                    user {
                        id
                        name
                        email
                    }
                }
            }
        ');
        
        $response->assertJsonPath('data.register.user.email', 'test@example.com');
        $this->assertNotEmpty($response->json('data.register.access_token'));
        $this->assertDatabaseHas('users', ['email' => 'test@example.com']);
    }
    
    public function test_cannot_create_product_without_auth(): void
    {
        $response = $this->graphQL('
            mutation {
                createProduct(input: {
                    name: "New Product"
                    description: "Test description"
                    price: 100
                    stock: 10
                    category_id: 1
                }) {
                    id
                }
            }
        ');
        
        $response->assertGraphQLErrorMessage('Unauthenticated.');
    }
    
    private function graphQL(string $query, array $variables = []): \Illuminate\Testing\TestResponse
    {
        return $this->postJson('/graphql', [
            'query' => $query,
            'variables' => $variables,
        ]);
    }
}
```

---

## Introspection & Documentation

```bash
# ดู Schema ทั้งหมด
curl -X POST http://localhost/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __schema { types { name } } }"}'

# ดู Type ที่ Specific
curl -X POST http://localhost/graphql \
  -H "Content-Type: application/json" \
  -d '{
    "query": "{ __type(name: \"Product\") { fields { name type { name } } } }"
  }'
```

---

## สรุป

| Feature | REST | GraphQL |
|---------|------|---------|
| Data fetching | Fixed response | Request ที่ต้องการ |
| Endpoints | หลาย Endpoints | Single Endpoint |
| Versioning | API Versions | Schema Evolution |
| Documentation | Manual/Swagger | Self-documenting |
| Real-time | WebSocket แยก | Built-in Subscriptions |
| Typing | Loose | Strongly Typed |
| Learning curve | ต่ำ | ปานกลาง |

---

*GraphQL ดีที่สุดเมื่อมี Client หลายชนิด (Mobile, Web, Desktop) ที่ต้องการข้อมูลต่างกัน*
