---
title: "SOLID Design Patterns in Laravel: Building Maintainable Applications"
date: 2024-11-12
draft: false
description: "Master SOLID principles in Laravel to write clean, maintainable, and scalable code. Learn practical implementations with real-world examples."
tags: ["Laravel", "PHP", "SOLID", "Design Patterns", "Architecture"]
categories: ["Web Development"]
---

## SOLID Design Patterns in Laravel: Building Maintainable Applications

SOLID principles are the foundation of clean, maintainable code. In this comprehensive guide, we'll explore how to apply each SOLID principle in Laravel applications with practical examples.

---

## What is SOLID?

SOLID is an acronym for five design principles that help developers create more maintainable and scalable software:

- **S** - Single Responsibility Principle
- **O** - Open/Closed Principle
- **L** - Liskov Substitution Principle
- **I** - Interface Segregation Principle
- **D** - Dependency Inversion Principle

---

## 1. Single Responsibility Principle (SRP)

**Definition:** A class should have only one reason to change, meaning it should have only one job or responsibility.

### ❌ Bad Example:

```php
class UserController extends Controller
{
    public function store(Request $request)
    {
        // Validation
        $validated = $request->validate([
            'name' => 'required|string',
            'email' => 'required|email|unique:users',
            'password' => 'required|min:8',
        ]);

        // Create user
        $user = User::create([
            'name' => $validated['name'],
            'email' => $validated['email'],
            'password' => Hash::make($validated['password']),
        ]);

        // Send welcome email
        Mail::to($user->email)->send(new WelcomeMail($user));

        // Log activity
        Log::info('New user registered', ['user_id' => $user->id]);

        return response()->json($user, 201);
    }
}
```

### ✅ Good Example:

```php
// Controller - handles HTTP requests only
class UserController extends Controller
{
    public function __construct(
        private UserService $userService
    ) {}

    public function store(StoreUserRequest $request)
    {
        $user = $this->userService->createUser($request->validated());
        
        return new UserResource($user);
    }
}

// Service - handles business logic
class UserService
{
    public function __construct(
        private UserRepository $userRepository,
        private NotificationService $notificationService,
        private ActivityLogger $activityLogger
    ) {}

    public function createUser(array $data): User
    {
        $user = $this->userRepository->create([
            'name' => $data['name'],
            'email' => $data['email'],
            'password' => Hash::make($data['password']),
        ]);

        $this->notificationService->sendWelcomeEmail($user);
        $this->activityLogger->logUserRegistration($user);

        return $user;
    }
}
```

---

## 2. Open/Closed Principle (OCP)

**Definition:** Software entities should be open for extension but closed for modification.

### ✅ Good Example Using Strategy Pattern:

```php
// Payment processor interface
interface PaymentProcessorInterface
{
    public function process(Order $order): PaymentResult;
}

// Concrete implementations
class StripePaymentProcessor implements PaymentProcessorInterface
{
    public function process(Order $order): PaymentResult
    {
        // Stripe-specific payment logic
        return new PaymentResult(true, 'stripe_transaction_id');
    }
}

class PayPalPaymentProcessor implements PaymentProcessorInterface
{
    public function process(Order $order): PaymentResult
    {
        // PayPal-specific payment logic
        return new PaymentResult(true, 'paypal_transaction_id');
    }
}

class CryptoPaymentProcessor implements PaymentProcessorInterface
{
    public function process(Order $order): PaymentResult
    {
        // Crypto-specific payment logic
        return new PaymentResult(true, 'crypto_transaction_id');
    }
}

// Payment service - open for extension, closed for modification
class PaymentService
{
    public function __construct(
        private PaymentProcessorInterface $processor
    ) {}

    public function processPayment(Order $order): PaymentResult
    {
        return $this->processor->process($order);
    }
}

// Usage in controller
public function processPayment(Request $request, Order $order)
{
    $processor = match($request->payment_method) {
        'stripe' => new StripePaymentProcessor(),
        'paypal' => new PayPalPaymentProcessor(),
        'crypto' => new CryptoPaymentProcessor(),
    };

    $paymentService = new PaymentService($processor);
    $result = $paymentService->processPayment($order);

    return response()->json($result);
}
```

---

## 3. Liskov Substitution Principle (LSP)

**Definition:** Objects of a superclass should be replaceable with objects of its subclasses without breaking the application.

### ✅ Good Example:

```php
abstract class StorageDriver
{
    abstract public function store(string $path, string $content): bool;
    abstract public function retrieve(string $path): ?string;
    abstract public function delete(string $path): bool;
}

class LocalStorageDriver extends StorageDriver
{
    public function store(string $path, string $content): bool
    {
        return Storage::disk('local')->put($path, $content);
    }

    public function retrieve(string $path): ?string
    {
        return Storage::disk('local')->get($path);
    }

    public function delete(string $path): bool
    {
        return Storage::disk('local')->delete($path);
    }
}

class S3StorageDriver extends StorageDriver
{
    public function store(string $path, string $content): bool
    {
        return Storage::disk('s3')->put($path, $content);
    }

    public function retrieve(string $path): ?string
    {
        return Storage::disk('s3')->get($path);
    }

    public function delete(string $path): bool
    {
        return Storage::disk('s3')->delete($path);
    }
}

// Service using storage - works with any StorageDriver implementation
class FileService
{
    public function __construct(
        private StorageDriver $storage
    ) {}

    public function uploadFile(UploadedFile $file): string
    {
        $path = 'uploads/' . $file->hashName();
        $this->storage->store($path, $file->getContent());
        return $path;
    }
}
```

---

## 4. Interface Segregation Principle (ISP)

**Definition:** Clients should not be forced to depend on interfaces they don't use.

### ❌ Bad Example:

```php
interface EmployeeInterface
{
    public function work(): void;
    public function eat(): void;
    public function calculateSalary(): float;
    public function manageTeam(): void;
}

// Problem: Regular employees don't manage teams
class RegularEmployee implements EmployeeInterface
{
    public function work(): void { /* ... */ }
    public function eat(): void { /* ... */ }
    public function calculateSalary(): float { /* ... */ }
    
    // Forced to implement this even though it doesn't apply
    public function manageTeam(): void 
    {
        throw new Exception('Not applicable');
    }
}
```

### ✅ Good Example:

```php
interface WorkableInterface
{
    public function work(): void;
}

interface EatableInterface
{
    public function eat(): void;
}

interface SalaryCalculableInterface
{
    public function calculateSalary(): float;
}

interface ManageableInterface
{
    public function manageTeam(): void;
}

class RegularEmployee implements 
    WorkableInterface, 
    EatableInterface, 
    SalaryCalculableInterface
{
    public function work(): void { /* ... */ }
    public function eat(): void { /* ... */ }
    public function calculateSalary(): float { /* ... */ }
}

class Manager implements 
    WorkableInterface, 
    EatableInterface, 
    SalaryCalculableInterface, 
    ManageableInterface
{
    public function work(): void { /* ... */ }
    public function eat(): void { /* ... */ }
    public function calculateSalary(): float { /* ... */ }
    public function manageTeam(): void { /* ... */ }
}
```

---

## 5. Dependency Inversion Principle (DIP)

**Definition:** High-level modules should not depend on low-level modules. Both should depend on abstractions.

### ❌ Bad Example:

```php
class UserController extends Controller
{
    public function store(Request $request)
    {
        // Direct dependency on concrete class
        $logger = new FileLogger();
        $cache = new RedisCache();
        
        $user = User::create($request->all());
        
        $logger->log('User created');
        $cache->put("user:{$user->id}", $user);
        
        return response()->json($user);
    }
}
```

### ✅ Good Example:

```php
// Abstractions
interface LoggerInterface
{
    public function log(string $message): void;
}

interface CacheInterface
{
    public function put(string $key, mixed $value): void;
    public function get(string $key): mixed;
}

// Concrete implementations
class FileLogger implements LoggerInterface
{
    public function log(string $message): void
    {
        Log::info($message);
    }
}

class RedisCache implements CacheInterface
{
    public function put(string $key, mixed $value): void
    {
        Cache::put($key, $value);
    }

    public function get(string $key): mixed
    {
        return Cache::get($key);
    }
}

// Service depending on abstractions
class UserService
{
    public function __construct(
        private LoggerInterface $logger,
        private CacheInterface $cache
    ) {}

    public function createUser(array $data): User
    {
        $user = User::create($data);
        
        $this->logger->log("User created: {$user->id}");
        $this->cache->put("user:{$user->id}", $user);
        
        return $user;
    }
}

// Controller with dependency injection
class UserController extends Controller
{
    public function __construct(
        private UserService $userService
    ) {}

    public function store(Request $request)
    {
        $user = $this->userService->createUser($request->validated());
        return new UserResource($user);
    }
}

// Service Provider binding
class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(LoggerInterface::class, FileLogger::class);
        $this->app->bind(CacheInterface::class, RedisCache::class);
    }
}
```

---

## Practical Implementation: Complete Example

Let's build a notification system using all SOLID principles:

```php
// Interfaces
interface NotificationChannelInterface
{
    public function send(User $user, string $message): bool;
}

interface NotificationFormatterInterface
{
    public function format(string $message): string;
}

// Concrete implementations
class EmailNotificationChannel implements NotificationChannelInterface
{
    public function send(User $user, string $message): bool
    {
        Mail::to($user->email)->send(new GenericNotification($message));
        return true;
    }
}

class SmsNotificationChannel implements NotificationChannelInterface
{
    public function send(User $user, string $message): bool
    {
        // SMS sending logic
        return true;
    }
}

class PushNotificationChannel implements NotificationChannelInterface
{
    public function send(User $user, string $message): bool
    {
        // Push notification logic
        return true;
    }
}

// Formatter
class HtmlFormatter implements NotificationFormatterInterface
{
    public function format(string $message): string
    {
        return "<p>{$message}</p>";
    }
}

class PlainTextFormatter implements NotificationFormatterInterface
{
    public function format(string $message): string
    {
        return strip_tags($message);
    }
}

// Service
class NotificationService
{
    public function __construct(
        private array $channels,
        private NotificationFormatterInterface $formatter
    ) {}

    public function notify(User $user, string $message): void
    {
        $formattedMessage = $this->formatter->format($message);

        foreach ($this->channels as $channel) {
            $channel->send($user, $formattedMessage);
        }
    }
}

// Usage
$notificationService = new NotificationService(
    channels: [
        new EmailNotificationChannel(),
        new SmsNotificationChannel(),
    ],
    formatter: new HtmlFormatter()
);

$notificationService->notify($user, 'Welcome to our platform!');
```

---

## Benefits of SOLID in Laravel

1. **Maintainability** - Code is easier to understand and modify
2. **Testability** - Components can be tested in isolation
3. **Scalability** - Easy to add new features without breaking existing code
4. **Flexibility** - Swap implementations without changing dependent code
5. **Reusability** - Components can be reused across the application

---

## Best Practices

1. **Use Laravel's Service Container** - Leverage dependency injection
2. **Create dedicated Service classes** - Keep controllers thin
3. **Use Form Requests** - Validate data outside controllers
4. **Implement Repository Pattern** - Abstract data access layer
5. **Use Action classes** - Single-purpose business logic
6. **Write Interfaces** - Define contracts for your implementations

---

## Conclusion

SOLID principles aren't just theoretical concepts—they're practical guidelines that lead to better Laravel applications. Start small, refactor incrementally, and you'll see significant improvements in code quality and maintainability.

Remember: **Good design is not about following rules blindly, but understanding when and how to apply them effectively.**
