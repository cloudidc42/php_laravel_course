# Part 89: Clean Architecture ใน PHP

## บทนำ

Clean Architecture เสนอโดย Robert C. Martin (Uncle Bob) เป็นการจัดระเบียบ Code ให้ Business Rules อยู่ที่ศูนย์กลาง และ Dependencies ชี้เข้าหา Center เสมอ

**เป้าหมาย**:
- Independent of Frameworks
- Testable โดยไม่ต้องการ UI, Database, Web Server
- Independent of UI
- Independent of Database
- Independent of External Agencies

---

## Hexagonal Architecture (Ports & Adapters)

Hexagonal Architecture หรือ Ports & Adapters เสนอโดย Alistair Cockburn แบ่ง Application เป็น "Inside" และ "Outside"

```
                    ┌─────────────────────────────────┐
                    │         External Systems         │
                    │   (DB, API, UI, Message Queue)  │
                    └──────────────┬──────────────────┘
                                   │ Adapters
                    ┌──────────────▼──────────────────┐
                    │            Ports                 │
                    │    (Interfaces / Contracts)      │
                    └──────────────┬──────────────────┘
                                   │
                    ┌──────────────▼──────────────────┐
                    │        Application Core          │
                    │   (Business Logic / Use Cases)   │
                    └─────────────────────────────────┘
```

### โครงสร้างไฟล์

```
src/
├── Domain/               # Enterprise Business Rules
│   ├── User/
│   │   ├── User.php           # Entity
│   │   ├── UserId.php         # Value Object
│   │   ├── Email.php          # Value Object
│   │   └── UserRepository.php # Port (Interface)
│   └── Order/
│       ├── Order.php
│       └── OrderRepository.php
├── Application/          # Application Business Rules
│   ├── User/
│   │   ├── RegisterUser/
│   │   │   ├── RegisterUserCommand.php
│   │   │   ├── RegisterUserHandler.php
│   │   │   └── RegisterUserResult.php
│   │   └── GetUser/
│   │       ├── GetUserQuery.php
│   │       └── GetUserHandler.php
│   └── Ports/            # Outgoing Ports
│       ├── MailerPort.php
│       └── CachePort.php
├── Infrastructure/       # Adapters (Driven)
│   ├── Persistence/
│   │   ├── Eloquent/
│   │   │   └── EloquentUserRepository.php
│   │   └── InMemory/
│   │       └── InMemoryUserRepository.php
│   ├── Mail/
│   │   ├── SmtpMailer.php
│   │   └── FakeMailer.php
│   └── Cache/
│       └── RedisCache.php
└── Interfaces/           # Adapters (Driving)
    ├── Http/
    │   ├── Controllers/
    │   └── Requests/
    ├── Console/
    │   └── Commands/
    └── Queue/
        └── Handlers/
```

---

## Clean Architecture Layers

### Layer 1: Domain (Enterprise Business Rules)

```php
<?php

namespace App\Domain\User;

// Value Objects
final class UserId
{
    public function __construct(private readonly string $value)
    {
        if (!preg_match('/^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/', $value)) {
            throw new \InvalidArgumentException("Invalid UUID format");
        }
    }
    
    public static function generate(): self
    {
        return new self(self::generateUUID());
    }
    
    public static function fromString(string $value): self
    {
        return new self($value);
    }
    
    private static function generateUUID(): string
    {
        $data = random_bytes(16);
        $data[6] = chr(ord($data[6]) & 0x0f | 0x40);
        $data[8] = chr(ord($data[8]) & 0x3f | 0x80);
        return vsprintf('%s%s-%s-%s-%s-%s%s%s', str_split(bin2hex($data), 4));
    }
    
    public function value(): string { return $this->value; }
    
    public function equals(UserId $other): bool
    {
        return $this->value === $other->value;
    }
    
    public function __toString(): string { return $this->value; }
}

final class Email
{
    private readonly string $value;
    
    public function __construct(string $email)
    {
        $normalized = strtolower(trim($email));
        if (!filter_var($normalized, FILTER_VALIDATE_EMAIL)) {
            throw new \InvalidArgumentException("Invalid email: {$email}");
        }
        $this->value = $normalized;
    }
    
    public function value(): string { return $this->value; }
    public function equals(Email $other): bool { return $this->value === $other->value; }
    public function __toString(): string { return $this->value; }
}

final class HashedPassword
{
    private string $hash;
    
    private function __construct(string $hash)
    {
        $this->hash = $hash;
    }
    
    public static function fromPlainText(string $plainText): self
    {
        if (strlen($plainText) < 8) {
            throw new \InvalidArgumentException("Password must be at least 8 characters");
        }
        return new self(password_hash($plainText, PASSWORD_ARGON2ID));
    }
    
    public static function fromHash(string $hash): self
    {
        return new self($hash);
    }
    
    public function verify(string $plainText): bool
    {
        return password_verify($plainText, $this->hash);
    }
    
    public function value(): string { return $this->hash; }
}

// Domain Entity
class User
{
    private array $domainEvents = [];
    private \DateTimeImmutable $createdAt;
    private ?\DateTimeImmutable $updatedAt = null;
    private bool $emailVerified = false;
    
    private function __construct(
        private readonly UserId $id,
        private string $name,
        private Email $email,
        private HashedPassword $password,
        private UserRole $role = UserRole::USER
    ) {
        $this->createdAt = new \DateTimeImmutable();
    }
    
    public static function register(
        string $name,
        Email $email,
        HashedPassword $password
    ): self {
        $user = new self(
            id: UserId::generate(),
            name: $name,
            email: $email,
            password: $password
        );
        
        $user->raise(new UserRegistered($user->id, $user->email, $user->name));
        
        return $user;
    }
    
    public static function reconstitute(
        UserId $id,
        string $name,
        Email $email,
        HashedPassword $password,
        UserRole $role,
        bool $emailVerified,
        \DateTimeImmutable $createdAt
    ): self {
        $user = new self($id, $name, $email, $password, $role);
        $user->emailVerified = $emailVerified;
        $user->createdAt = $createdAt;
        return $user;
    }
    
    public function verifyEmail(): void
    {
        if ($this->emailVerified) {
            throw new \DomainException("Email already verified");
        }
        
        $this->emailVerified = true;
        $this->touch();
        $this->raise(new UserEmailVerified($this->id));
    }
    
    public function changeEmail(Email $newEmail): void
    {
        if ($this->email->equals($newEmail)) return;
        
        $this->email = $newEmail;
        $this->emailVerified = false;
        $this->touch();
        
        $this->raise(new UserEmailChanged($this->id, $newEmail));
    }
    
    public function changePassword(string $currentPassword, string $newPassword): void
    {
        if (!$this->password->verify($currentPassword)) {
            throw new \DomainException("Current password is incorrect");
        }
        
        $this->password = HashedPassword::fromPlainText($newPassword);
        $this->touch();
        
        $this->raise(new UserPasswordChanged($this->id));
    }
    
    public function promoteToAdmin(): void
    {
        if ($this->role === UserRole::ADMIN) {
            throw new \DomainException("User is already an admin");
        }
        
        $this->role = UserRole::ADMIN;
        $this->touch();
        
        $this->raise(new UserPromotedToAdmin($this->id));
    }
    
    private function touch(): void
    {
        $this->updatedAt = new \DateTimeImmutable();
    }
    
    private function raise(\DomainEvent $event): void
    {
        $this->domainEvents[] = $event;
    }
    
    public function pullEvents(): array
    {
        $events = $this->domainEvents;
        $this->domainEvents = [];
        return $events;
    }
    
    // Getters
    public function getId(): UserId { return $this->id; }
    public function getName(): string { return $this->name; }
    public function getEmail(): Email { return $this->email; }
    public function getPassword(): HashedPassword { return $this->password; }
    public function getRole(): UserRole { return $this->role; }
    public function isEmailVerified(): bool { return $this->emailVerified; }
    public function getCreatedAt(): \DateTimeImmutable { return $this->createdAt; }
}

// Domain Port (Outgoing)
interface UserRepository
{
    public function findById(UserId $id): ?User;
    public function findByEmail(Email $email): ?User;
    public function save(User $user): void;
    public function exists(Email $email): bool;
    public function delete(UserId $id): void;
}

// Domain Service
class UserAuthenticationService
{
    public function authenticate(User $user, string $password): bool
    {
        if (!$user->isEmailVerified()) {
            throw new \DomainException("Email not verified");
        }
        
        return $user->getPassword()->verify($password);
    }
}
```

---

### Layer 2: Application (Application Business Rules)

```php
<?php

namespace App\Application\User\RegisterUser;

// Command
class RegisterUserCommand
{
    public function __construct(
        public readonly string $name,
        public readonly string $email,
        public readonly string $password,
        public readonly string $passwordConfirmation
    ) {}
}

// Result
class RegisterUserResult
{
    public function __construct(
        public readonly string $userId,
        public readonly string $email,
        public readonly string $name
    ) {}
}

// Application Port (Outgoing)
interface MailerPort
{
    public function sendWelcomeEmail(string $email, string $name): void;
    public function sendVerificationEmail(string $email, string $token): void;
}

interface CachePort
{
    public function set(string $key, mixed $value, int $ttl = 3600): void;
    public function get(string $key): mixed;
    public function delete(string $key): void;
}

interface EventBusPort
{
    public function publish(\DomainEvent $event): void;
}

// Use Case Handler
class RegisterUserHandler
{
    public function __construct(
        private readonly UserRepository $users,
        private readonly MailerPort $mailer,
        private readonly EventBusPort $eventBus
    ) {}
    
    public function handle(RegisterUserCommand $command): RegisterUserResult
    {
        // Validate
        if ($command->password !== $command->passwordConfirmation) {
            throw new \InvalidArgumentException("Passwords do not match");
        }
        
        $email = new Email($command->email);
        
        // Check uniqueness
        if ($this->users->exists($email)) {
            throw new \DomainException("Email already registered: {$email}");
        }
        
        // Create user
        $user = User::register(
            name: $command->name,
            email: $email,
            password: HashedPassword::fromPlainText($command->password)
        );
        
        // Persist
        $this->users->save($user);
        
        // Publish events
        foreach ($user->pullEvents() as $event) {
            $this->eventBus->publish($event);
        }
        
        // Send welcome email
        $this->mailer->sendWelcomeEmail($email->value(), $command->name);
        
        return new RegisterUserResult(
            userId: $user->getId()->value(),
            email: $email->value(),
            name: $command->name
        );
    }
}

// Query Handler
namespace App\Application\User\GetUser;

class GetUserQuery
{
    public function __construct(public readonly string $userId) {}
}

class GetUserDTO
{
    public function __construct(
        public readonly string $id,
        public readonly string $name,
        public readonly string $email,
        public readonly string $role,
        public readonly bool $emailVerified,
        public readonly string $createdAt
    ) {}
}

class GetUserHandler
{
    public function __construct(private readonly UserRepository $users) {}
    
    public function handle(GetUserQuery $query): GetUserDTO
    {
        $user = $this->users->findById(UserId::fromString($query->userId));
        
        if (!$user) {
            throw new \DomainException("User not found: {$query->userId}");
        }
        
        return new GetUserDTO(
            id: $user->getId()->value(),
            name: $user->getName(),
            email: $user->getEmail()->value(),
            role: $user->getRole()->value,
            emailVerified: $user->isEmailVerified(),
            createdAt: $user->getCreatedAt()->format(\DateTime::ISO8601)
        );
    }
}
```

---

### Layer 3: Infrastructure (Adapters - Driven Side)

```php
<?php

namespace App\Infrastructure\Persistence\Eloquent;

use App\Domain\User\{User, UserId, Email, HashedPassword, UserRepository, UserRole};
use App\Models\User as UserModel;

class EloquentUserRepository implements UserRepository
{
    public function findById(UserId $id): ?User
    {
        $model = UserModel::find($id->value());
        return $model ? $this->toDomain($model) : null;
    }
    
    public function findByEmail(Email $email): ?User
    {
        $model = UserModel::where('email', $email->value())->first();
        return $model ? $this->toDomain($model) : null;
    }
    
    public function save(User $user): void
    {
        UserModel::updateOrCreate(
            ['id' => $user->getId()->value()],
            [
                'name' => $user->getName(),
                'email' => $user->getEmail()->value(),
                'password' => $user->getPassword()->value(),
                'role' => $user->getRole()->value,
                'email_verified' => $user->isEmailVerified(),
                'created_at' => $user->getCreatedAt(),
            ]
        );
    }
    
    public function exists(Email $email): bool
    {
        return UserModel::where('email', $email->value())->exists();
    }
    
    public function delete(UserId $id): void
    {
        UserModel::where('id', $id->value())->delete();
    }
    
    private function toDomain(UserModel $model): User
    {
        return User::reconstitute(
            id: UserId::fromString($model->id),
            name: $model->name,
            email: new Email($model->email),
            password: HashedPassword::fromHash($model->password),
            role: UserRole::from($model->role),
            emailVerified: (bool)$model->email_verified,
            createdAt: new \DateTimeImmutable($model->created_at)
        );
    }
}

// In-Memory Repository (for testing)
class InMemoryUserRepository implements UserRepository
{
    private array $users = [];
    
    public function findById(UserId $id): ?User
    {
        return $this->users[$id->value()] ?? null;
    }
    
    public function findByEmail(Email $email): ?User
    {
        foreach ($this->users as $user) {
            if ($user->getEmail()->equals($email)) {
                return $user;
            }
        }
        return null;
    }
    
    public function save(User $user): void
    {
        $this->users[$user->getId()->value()] = $user;
    }
    
    public function exists(Email $email): bool
    {
        return $this->findByEmail($email) !== null;
    }
    
    public function delete(UserId $id): void
    {
        unset($this->users[$id->value()]);
    }
    
    public function findAll(): array
    {
        return array_values($this->users);
    }
}

// Mail Adapter
namespace App\Infrastructure\Mail;

class LaravelMailerAdapter implements \App\Application\Ports\MailerPort
{
    public function sendWelcomeEmail(string $email, string $name): void
    {
        \App\Mail\WelcomeMail::dispatch($email, $name);
    }
    
    public function sendVerificationEmail(string $email, string $token): void
    {
        \App\Mail\VerifyEmailMail::dispatch($email, $token);
    }
}

class LogMailerAdapter implements \App\Application\Ports\MailerPort
{
    private array $sentEmails = [];
    
    public function sendWelcomeEmail(string $email, string $name): void
    {
        $this->sentEmails[] = ['type' => 'welcome', 'to' => $email, 'name' => $name];
    }
    
    public function sendVerificationEmail(string $email, string $token): void
    {
        $this->sentEmails[] = ['type' => 'verify', 'to' => $email, 'token' => $token];
    }
    
    public function getSentEmails(): array
    {
        return $this->sentEmails;
    }
}
```

---

### Layer 4: Interfaces (Adapters - Driving Side)

```php
<?php

namespace App\Interfaces\Http\Controllers;

use App\Application\User\RegisterUser\{RegisterUserCommand, RegisterUserHandler};
use App\Application\User\GetUser\{GetUserQuery, GetUserHandler};
use Illuminate\Http\{Request, JsonResponse};

class UserController
{
    public function __construct(
        private readonly RegisterUserHandler $registerHandler,
        private readonly GetUserHandler $getUserHandler
    ) {}
    
    public function register(Request $request): JsonResponse
    {
        $request->validate([
            'name' => 'required|string|min:2|max:100',
            'email' => 'required|email',
            'password' => 'required|string|min:8',
            'password_confirmation' => 'required|string',
        ]);
        
        try {
            $result = $this->registerHandler->handle(
                new RegisterUserCommand(
                    name: $request->name,
                    email: $request->email,
                    password: $request->password,
                    passwordConfirmation: $request->password_confirmation
                )
            );
            
            return response()->json([
                'success' => true,
                'data' => [
                    'id' => $result->userId,
                    'email' => $result->email,
                    'name' => $result->name,
                ]
            ], 201);
            
        } catch (\DomainException $e) {
            return response()->json([
                'success' => false,
                'message' => $e->getMessage()
            ], 422);
        }
    }
    
    public function show(string $userId): JsonResponse
    {
        try {
            $user = $this->getUserHandler->handle(new GetUserQuery($userId));
            
            return response()->json([
                'success' => true,
                'data' => $user
            ]);
        } catch (\DomainException $e) {
            return response()->json([
                'success' => false,
                'message' => $e->getMessage()
            ], 404);
        }
    }
}

// Console Adapter
namespace App\Interfaces\Console;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\{InputInterface, InputArgument};
use Symfony\Component\Console\Output\OutputInterface;
use App\Application\User\RegisterUser\{RegisterUserCommand, RegisterUserHandler};

class CreateAdminUserCommand extends Command
{
    protected static $defaultName = 'users:create-admin';
    
    public function __construct(private RegisterUserHandler $handler)
    {
        parent::__construct();
    }
    
    protected function configure(): void
    {
        $this->addArgument('name', InputArgument::REQUIRED)
             ->addArgument('email', InputArgument::REQUIRED)
             ->addArgument('password', InputArgument::REQUIRED);
    }
    
    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $password = $input->getArgument('password');
        
        try {
            $result = $this->handler->handle(new RegisterUserCommand(
                name: $input->getArgument('name'),
                email: $input->getArgument('email'),
                password: $password,
                passwordConfirmation: $password
            ));
            
            $output->writeln("<info>Admin user created: {$result->userId}</info>");
            return Command::SUCCESS;
            
        } catch (\Exception $e) {
            $output->writeln("<error>{$e->getMessage()}</error>");
            return Command::FAILURE;
        }
    }
}
```

---

## Dependency Rule

```
Outer layers depend on Inner layers - NEVER the other way around

┌─────────────────────────────────────────────┐
│  Interfaces / Infrastructure (Outer)        │
│  ┌──────────────────────────────────────┐   │
│  │  Application Layer                   │   │
│  │  ┌─────────────────────────────────┐ │   │
│  │  │  Domain Layer (Inner)           │ │   │
│  │  │                                 │ │   │
│  │  └─────────────────────────────────┘ │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘

Dependency: → (depends on)
Infrastructure → Application → Domain
Interfaces → Application → Domain

Domain ไม่รู้จักชั้นไหนทั้งนั้น
```

---

## Workshop: สร้าง Clean Architecture Project

### Step 1: Project Structure

```bash
mkdir -p src/{Domain,Application,Infrastructure,Interfaces}
mkdir -p src/Domain/{User,Order,Product}/
mkdir -p src/Application/{User,Order}/{Commands,Queries,Ports}/
mkdir -p src/Infrastructure/{Persistence,Mail,Cache,Queue}/
mkdir -p src/Interfaces/{Http,Console,Queue}/
mkdir -p tests/{Unit,Integration,Feature}/
```

### Step 2: Dependency Injection Container

```php
<?php

namespace App\Infrastructure\Container;

use Illuminate\Support\ServiceProvider;
use App\Domain\User\UserRepository;
use App\Infrastructure\Persistence\Eloquent\EloquentUserRepository;
use App\Application\Ports\MailerPort;
use App\Infrastructure\Mail\LaravelMailerAdapter;
use App\Application\User\RegisterUser\RegisterUserHandler;
use App\Application\User\GetUser\GetUserHandler;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Domain -> Infrastructure bindings
        $this->app->bind(UserRepository::class, EloquentUserRepository::class);
        $this->app->bind(MailerPort::class, LaravelMailerAdapter::class);
        
        // Application Services
        $this->app->bind(RegisterUserHandler::class, function ($app) {
            return new RegisterUserHandler(
                users: $app->make(UserRepository::class),
                mailer: $app->make(MailerPort::class),
                eventBus: $app->make(EventBusPort::class)
            );
        });
        
        $this->app->bind(GetUserHandler::class, function ($app) {
            return new GetUserHandler(
                users: $app->make(UserRepository::class)
            );
        });
    }
}
```

### Step 3: Testing

```php
<?php

namespace Tests\Unit\Application;

use App\Application\User\RegisterUser\{RegisterUserCommand, RegisterUserHandler};
use App\Infrastructure\Persistence\InMemory\InMemoryUserRepository;
use App\Infrastructure\Mail\LogMailerAdapter;
use PHPUnit\Framework\TestCase;

class RegisterUserHandlerTest extends TestCase
{
    private RegisterUserHandler $handler;
    private InMemoryUserRepository $users;
    private LogMailerAdapter $mailer;
    private FakeEventBus $eventBus;
    
    protected function setUp(): void
    {
        $this->users = new InMemoryUserRepository();
        $this->mailer = new LogMailerAdapter();
        $this->eventBus = new FakeEventBus();
        
        $this->handler = new RegisterUserHandler(
            users: $this->users,
            mailer: $this->mailer,
            eventBus: $this->eventBus
        );
    }
    
    public function test_register_user_successfully(): void
    {
        $command = new RegisterUserCommand(
            name: 'John Doe',
            email: 'john@example.com',
            password: 'SecurePass123',
            passwordConfirmation: 'SecurePass123'
        );
        
        $result = $this->handler->handle($command);
        
        $this->assertNotEmpty($result->userId);
        $this->assertEquals('john@example.com', $result->email);
        $this->assertEquals('John Doe', $result->name);
        
        // Verify user saved
        $savedUser = $this->users->findByEmail(new Email('john@example.com'));
        $this->assertNotNull($savedUser);
        
        // Verify email sent
        $sentEmails = $this->mailer->getSentEmails();
        $this->assertCount(1, $sentEmails);
        $this->assertEquals('welcome', $sentEmails[0]['type']);
        
        // Verify event published
        $events = $this->eventBus->getPublishedEvents();
        $this->assertCount(1, $events);
        $this->assertInstanceOf(UserRegistered::class, $events[0]);
    }
    
    public function test_throws_when_email_already_exists(): void
    {
        // Register first user
        $this->handler->handle(new RegisterUserCommand(
            name: 'John Doe',
            email: 'john@example.com',
            password: 'SecurePass123',
            passwordConfirmation: 'SecurePass123'
        ));
        
        // Try to register with same email
        $this->expectException(\DomainException::class);
        $this->expectExceptionMessageMatches('/already registered/');
        
        $this->handler->handle(new RegisterUserCommand(
            name: 'Jane Doe',
            email: 'john@example.com',
            password: 'SecurePass456',
            passwordConfirmation: 'SecurePass456'
        ));
    }
    
    public function test_throws_when_passwords_dont_match(): void
    {
        $this->expectException(\InvalidArgumentException::class);
        $this->expectExceptionMessage("Passwords do not match");
        
        $this->handler->handle(new RegisterUserCommand(
            name: 'John Doe',
            email: 'john@example.com',
            password: 'SecurePass123',
            passwordConfirmation: 'DifferentPass456'
        ));
    }
}

// Fake Implementations for Testing
class FakeEventBus implements EventBusPort
{
    private array $events = [];
    
    public function publish(\DomainEvent $event): void
    {
        $this->events[] = $event;
    }
    
    public function getPublishedEvents(): array
    {
        return $this->events;
    }
}
```

---

## Use Cases Pattern

```php
<?php

// Command Bus Pattern
interface CommandBus
{
    public function dispatch(object $command): mixed;
}

class SimpleCommandBus implements CommandBus
{
    private array $handlers = [];
    private array $middlewares = [];
    
    public function register(string $commandClass, callable $handler): void
    {
        $this->handlers[$commandClass] = $handler;
    }
    
    public function addMiddleware(callable $middleware): void
    {
        $this->middlewares[] = $middleware;
    }
    
    public function dispatch(object $command): mixed
    {
        $handler = $this->handlers[get_class($command)]
            ?? throw new \RuntimeException("No handler for " . get_class($command));
        
        // Apply middlewares
        $next = $handler;
        foreach (array_reverse($this->middlewares) as $middleware) {
            $next = fn($cmd) => $middleware($cmd, $next);
        }
        
        return $next($command);
    }
}

// Middlewares
class LoggingMiddleware
{
    public function __invoke(object $command, callable $next): mixed
    {
        $commandName = get_class($command);
        echo "Executing: {$commandName}\n";
        
        $start = microtime(true);
        $result = $next($command);
        $elapsed = round((microtime(true) - $start) * 1000, 2);
        
        echo "Completed: {$commandName} in {$elapsed}ms\n";
        
        return $result;
    }
}

class TransactionalMiddleware
{
    public function __construct(private \PDO $db) {}
    
    public function __invoke(object $command, callable $next): mixed
    {
        $this->db->beginTransaction();
        
        try {
            $result = $next($command);
            $this->db->commit();
            return $result;
        } catch (\Exception $e) {
            $this->db->rollBack();
            throw $e;
        }
    }
}

// Setup
$bus = new SimpleCommandBus();
$bus->addMiddleware(new LoggingMiddleware());
$bus->addMiddleware(new TransactionalMiddleware($pdo));

$bus->register(RegisterUserCommand::class, fn($cmd) => $registerHandler->handle($cmd));
$bus->register(PlaceOrderCommand::class, fn($cmd) => $orderHandler->handle($cmd));

// Usage
$result = $bus->dispatch(new RegisterUserCommand(
    name: 'สมชาย',
    email: 'somchai@example.com',
    password: 'MyPassword123',
    passwordConfirmation: 'MyPassword123'
));
```

---

## สรุป

| Layer | ชื่อ | รับผิดชอบ | Dependencies |
|-------|------|---------|-------------|
| Innermost | Domain | Business Rules, Entities | ไม่ขึ้นกับใคร |
| 2nd | Application | Use Cases, Orchestration | Domain only |
| 3rd | Infrastructure | DB, Mail, Cache | Domain, Application |
| Outermost | Interfaces | HTTP, Console, Queue | Application |

**กฎสำคัญ**: Dependencies ต้องชี้เข้าหา Center เสมอ (Dependency Rule)

---

*Clean Architecture ช่วยให้เปลี่ยน Database, Framework, หรือ External Service ได้โดยไม่กระทบ Business Logic*
