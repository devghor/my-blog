---
title: "Laravel High-Concurrency Applications: Performance & Scalability"
date: 2024-11-08
draft: false
description: "Build high-performance Laravel applications that handle thousands of concurrent requests. Learn optimization techniques, caching strategies, and scaling patterns."
tags: ["Laravel", "Performance", "Scalability", "Optimization", "Architecture"]
categories: ["Software Architecture"]
---

## Laravel High-Concurrency Applications: Performance & Scalability

Building applications that handle high concurrency requires careful planning and optimization. In this guide, we'll explore proven techniques to make your Laravel applications blazingly fast and scalable.

---

## Understanding High Concurrency

High-concurrency applications must handle:
- Thousands of simultaneous requests
- Real-time data processing
- Heavy database operations
- Large-scale data transfers

---

## 1. Database Optimization

### Query Optimization

```php
// ❌ N+1 Problem
$users = User::all();
foreach ($users as $user) {
    echo $user->posts->count(); // Queries for each user
}

// ✅ Eager Loading
$users = User::with('posts')->get();
foreach ($users as $user) {
    echo $user->posts->count(); // No additional queries
}

// ✅ Load Only Required Columns
$users = User::select('id', 'name', 'email')
    ->with('posts:id,user_id,title')
    ->get();

// ✅ Chunk Large Datasets
User::chunk(1000, function ($users) {
    foreach ($users as $user) {
        // Process user
    }
});
```

### Database Indexing

```php
// Migration
Schema::create('orders', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->index(); // Index foreign keys
    $table->string('status')->index(); // Index frequently queried columns
    $table->timestamp('created_at')->index();
    
    // Composite index for common queries
    $table->index(['user_id', 'status']);
    $table->index(['status', 'created_at']);
});
```

### Connection Pooling

```php
// config/database.php
'mysql' => [
    'read' => [
        'host' => [
            '192.168.1.1', // Read replica 1
            '192.168.1.2', // Read replica 2
        ],
    ],
    'write' => [
        'host' => ['192.168.1.3'], // Master
    ],
    'sticky' => true, // Read from master after write
    'pool' => [
        'min' => 2,
        'max' => 10,
    ],
],
```

---

## 2. Caching Strategies

### Multi-Layer Caching

```php
class ProductService
{
    public function getProduct(int $id): ?Product
    {
        // Layer 1: Application cache (APCu/Array)
        $cacheKey = "product:{$id}";
        
        if ($cached = apcu_fetch($cacheKey)) {
            return unserialize($cached);
        }

        // Layer 2: Redis cache
        $product = Cache::remember($cacheKey, 3600, function () use ($id) {
            // Layer 3: Database
            return Product::with(['category', 'images'])->find($id);
        });

        // Store in APCu for faster subsequent access
        apcu_store($cacheKey, serialize($product), 600);

        return $product;
    }
}
```

### Query Result Caching

```php
class OrderRepository
{
    public function getUserOrders(int $userId): Collection
    {
        return Cache::tags(['users', "user:{$userId}", 'orders'])
            ->remember("user:{$userId}:orders", 1800, function () use ($userId) {
                return Order::where('user_id', $userId)
                    ->with('items.product')
                    ->latest()
                    ->get();
            });
    }

    public function clearUserCache(int $userId): void
    {
        Cache::tags(["user:{$userId}"])->flush();
    }
}
```

### Cache Warming

```php
// Console Command
class WarmCacheCommand extends Command
{
    protected $signature = 'cache:warm';

    public function handle(): void
    {
        // Warm popular products
        Product::popular()->chunk(100, function ($products) {
            foreach ($products as $product) {
                Cache::put(
                    "product:{$product->id}",
                    $product->load('category', 'images'),
                    3600
                );
            }
        });

        $this->info('Cache warmed successfully!');
    }
}
```

---

## 3. Queue System for Background Processing

### Offload Heavy Tasks

```php
// Job
class ProcessLargeDataset implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public $timeout = 300;
    public $tries = 3;

    public function __construct(
        private int $userId,
        private array $data
    ) {}

    public function handle(): void
    {
        // Heavy processing here
        DB::table('analytics')->insert($this->data);
    }

    public function failed(Throwable $exception): void
    {
        // Handle failure
        Log::error('Dataset processing failed', [
            'user_id' => $this->userId,
            'error' => $exception->getMessage(),
        ]);
    }
}

// Controller
public function processData(Request $request)
{
    ProcessLargeDataset::dispatch(
        auth()->id(),
        $request->validated()
    );

    return response()->json(['message' => 'Processing started']);
}
```

### Batch Processing

```php
$batch = Bus::batch([
    new ProcessDataChunk($chunk1),
    new ProcessDataChunk($chunk2),
    new ProcessDataChunk($chunk3),
])->then(function (Batch $batch) {
    // All jobs completed successfully
    Cache::put('batch:' . $batch->id, 'completed');
})->catch(function (Batch $batch, Throwable $e) {
    // First failure detected
})->finally(function (Batch $batch) {
    // Batch finished executing
})->dispatch();

return response()->json(['batch_id' => $batch->id]);
```

---

## 4. API Rate Limiting & Throttling

### Custom Rate Limiter

```php
// RouteServiceProvider
protected function configureRateLimiting()
{
    RateLimiter::for('api', function (Request $request) {
        return Limit::perMinute(60)
            ->by($request->user()?->id ?: $request->ip())
            ->response(function (Request $request, array $headers) {
                return response()->json([
                    'message' => 'Too many requests'
                ], 429, $headers);
            });
    });

    // Premium users get higher limits
    RateLimiter::for('premium-api', function (Request $request) {
        if ($request->user()?->isPremium()) {
            return Limit::perMinute(1000);
        }
        
        return Limit::perMinute(60);
    });
}

// Route
Route::middleware(['auth:sanctum', 'throttle:premium-api'])
    ->get('/data', [DataController::class, 'index']);
```

---

## 5. Horizontal Scaling

### Load Balancing Configuration

```nginx
# nginx.conf
upstream laravel_backend {
    least_conn; # Load balancing method
    
    server 192.168.1.10:8000 weight=3;
    server 192.168.1.11:8000 weight=3;
    server 192.168.1.12:8000 weight=2;
    
    # Health checks
    keepalive 32;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://laravel_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### Session Management for Scaling

```php
// config/session.php
'driver' => 'redis',
'connection' => 'session',

// config/database.php
'redis' => [
    'session' => [
        'url' => env('REDIS_SESSION_URL'),
        'host' => env('REDIS_SESSION_HOST', '127.0.0.1'),
        'password' => env('REDIS_SESSION_PASSWORD'),
        'port' => env('REDIS_SESSION_PORT', 6379),
        'database' => 1,
    ],
],
```

---

## 6. Asset Optimization

### CDN Integration

```php
// config/filesystems.php
'cloudfront' => [
    'driver' => 's3',
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION'),
    'bucket' => env('AWS_BUCKET'),
    'url' => env('AWS_CLOUDFRONT_URL'), // CDN URL
],

// Usage
public function uploadImage(UploadedFile $file): string
{
    $path = $file->store('images', 'cloudfront');
    return Storage::disk('cloudfront')->url($path);
}
```

### Image Optimization

```php
use Intervention\Image\Facades\Image;

class ImageOptimizationService
{
    public function optimizeAndStore(UploadedFile $file): array
    {
        $image = Image::make($file);
        
        $versions = [
            'thumbnail' => ['width' => 150, 'height' => 150],
            'medium' => ['width' => 600, 'height' => 600],
            'large' => ['width' => 1200, 'height' => 1200],
        ];

        $paths = [];
        foreach ($versions as $size => $dimensions) {
            $resized = $image->fit($dimensions['width'], $dimensions['height']);
            $path = "images/{$size}/" . Str::uuid() . '.webp';
            
            Storage::put(
                $path,
                $resized->encode('webp', 80)
            );
            
            $paths[$size] = Storage::url($path);
        }

        return $paths;
    }
}
```

---

## 7. Monitoring & Profiling

### Performance Monitoring

```php
// Middleware
class PerformanceMonitor
{
    public function handle(Request $request, Closure $next)
    {
        $start = microtime(true);
        $startMemory = memory_get_usage();

        $response = $next($request);

        $duration = (microtime(true) - $start) * 1000;
        $memoryUsed = (memory_get_usage() - $startMemory) / 1024 / 1024;

        if ($duration > 1000) { // Log slow requests
            Log::warning('Slow request detected', [
                'url' => $request->fullUrl(),
                'method' => $request->method(),
                'duration_ms' => round($duration, 2),
                'memory_mb' => round($memoryUsed, 2),
            ]);
        }

        $response->header('X-Response-Time', round($duration, 2) . 'ms');

        return $response;
    }
}
```

### Query Monitoring

```php
// AppServiceProvider
public function boot(): void
{
    if (app()->environment('local')) {
        DB::listen(function ($query) {
            if ($query->time > 100) { // Log queries over 100ms
                Log::warning('Slow query detected', [
                    'sql' => $query->sql,
                    'bindings' => $query->bindings,
                    'time' => $query->time,
                ]);
            }
        });
    }
}
```

---

## 8. Real-Time Features with Broadcasting

### Efficient Broadcasting

```php
// Event
class OrderStatusUpdated implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(
        public Order $order
    ) {}

    public function broadcastOn(): array
    {
        return [
            new PrivateChannel('orders.' . $this->order->user_id),
        ];
    }

    public function broadcastWith(): array
    {
        return [
            'id' => $this->order->id,
            'status' => $this->order->status,
            'updated_at' => $this->order->updated_at,
        ];
    }

    public function broadcastQueue(): string
    {
        return 'broadcasts'; // Separate queue for broadcasts
    }
}

// config/broadcasting.php
'redis' => [
    'driver' => 'redis',
    'connection' => 'default',
    'queue' => 'broadcasts',
],
```

---

## Performance Benchmarks

### Before Optimization
- **Response Time:** 800ms
- **Requests/sec:** 50
- **Database Queries:** 45 per request
- **Memory Usage:** 128MB

### After Optimization
- **Response Time:** 80ms (10x improvement)
- **Requests/sec:** 500 (10x improvement)
- **Database Queries:** 3 per request
- **Memory Usage:** 32MB (4x improvement)

---

## Best Practices Checklist

- ✅ Use eager loading to prevent N+1 queries
- ✅ Implement multi-layer caching (Redis + APCu)
- ✅ Index database columns properly
- ✅ Use queue workers for heavy tasks
- ✅ Implement rate limiting on APIs
- ✅ Configure read/write database replicas
- ✅ Use CDN for static assets
- ✅ Monitor and log slow queries
- ✅ Optimize images before storage
- ✅ Use Redis for sessions and cache
- ✅ Implement horizontal scaling
- ✅ Regular performance profiling

---

## Conclusion

Building high-concurrency Laravel applications requires a holistic approach combining database optimization, caching strategies, background processing, and proper scaling techniques. Start with these optimizations and continuously monitor your application's performance to identify bottlenecks.

Remember: **Premature optimization is the root of all evil. Profile first, optimize second.**
