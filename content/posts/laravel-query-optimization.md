---
title: "Query Optimization in Laravel: A Complete Guide"
date: 2024-11-02
draft: false
description: "Master database query optimization in Laravel. Learn to eliminate N+1 queries, use indexes effectively, and boost application performance dramatically."
tags: ["Laravel", "Database", "Optimization", "Performance", "MySQL"]
categories: ["Web Development"]
---

## Query Optimization in Laravel: A Complete Guide

Slow queries can cripple your application's performance. This comprehensive guide will teach you how to identify, analyze, and optimize database queries in Laravel for maximum performance.

---

## 1. The N+1 Query Problem

The N+1 problem is the most common performance issue in Laravel applications.

### Problem Example

```php
// ❌ N+1 Query Problem
// This executes 1 query to get posts + N queries to get authors
$posts = Post::all(); // 1 query

foreach ($posts as $post) {
    echo $post->author->name; // N queries (one for each post)
}

// Total: 1 + N queries
```

### Solution: Eager Loading

```php
// ✅ Eager Loading - Only 2 queries
$posts = Post::with('author')->get(); // 2 queries total

foreach ($posts as $post) {
    echo $post->author->name; // No additional queries
}

// ✅ Multiple relationships
$posts = Post::with(['author', 'comments', 'tags'])->get();

// ✅ Nested relationships
$posts = Post::with('author.profile')->get();

// ✅ Conditional eager loading
$posts = Post::with(['comments' => function ($query) {
    $query->where('approved', true)
          ->orderBy('created_at', 'desc')
          ->limit(10);
}])->get();
```

### Lazy Eager Loading

```php
// When you forgot to eager load
$posts = Post::all();

// ✅ Load relationships after the fact
$posts->load('author');

// ✅ Load if not already loaded
$posts->loadMissing('comments');

// ✅ Load count without loading all data
$posts->loadCount('comments');
```

---

## 2. Select Only Required Columns

```php
// ❌ Fetching all columns (inefficient)
$users = User::all();

// ✅ Select only needed columns
$users = User::select('id', 'name', 'email')->get();

// ✅ Add columns later if needed
$users = User::select('id', 'name')
    ->addSelect('email')
    ->get();

// ✅ With relationships
$posts = Post::select('id', 'title', 'user_id')
    ->with('author:id,name,email')
    ->get();

// ✅ Exclude columns
$users = User::select('*')
    ->selectRaw('DATE(created_at) as date')
    ->get();
```

---

## 3. Database Indexing

### Creating Indexes

```php
// Migration
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->text('content');
    $table->foreignId('user_id');
    $table->string('status');
    $table->timestamp('published_at')->nullable();
    $table->timestamps();

    // ✅ Single column indexes
    $table->index('user_id');
    $table->index('status');
    $table->index('published_at');

    // ✅ Composite indexes (order matters!)
    $table->index(['user_id', 'status']);
    $table->index(['status', 'published_at']);

    // ✅ Unique indexes
    $table->unique('slug');

    // ✅ Full-text indexes (MySQL 5.7+)
    $table->fullText(['title', 'content']);
});

// Add index to existing table
Schema::table('posts', function (Blueprint $table) {
    $table->index('category_id');
});
```

### When to Use Indexes

```php
// ✅ Good candidates for indexing:
// - Foreign keys
// - Columns in WHERE clauses
// - Columns in ORDER BY
// - Columns in JOIN conditions
// - Columns frequently searched

// ❌ Don't index:
// - Small tables (< 1000 rows)
// - Columns with low cardinality (few unique values)
// - Columns that are frequently updated
// - Very large text columns
```

### Using Full-Text Search

```php
// Migration with full-text index
Schema::create('articles', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->text('content');
    $table->fullText(['title', 'content']);
});

// ✅ Full-text search query
$articles = Article::whereFullText(['title', 'content'], 'laravel optimization')
    ->get();

// ✅ Boolean mode for advanced searches
$articles = Article::whereFullText(
    ['title', 'content'],
    '+laravel +optimization -tutorial',
    ['mode' => 'boolean']
)->get();
```

---

## 4. Query Optimization Techniques

### Chunking Large Datasets

```php
// ❌ Memory intensive for large datasets
$users = User::all(); // Loads all users into memory

// ✅ Process in chunks
User::chunk(1000, function ($users) {
    foreach ($users as $user) {
        // Process user
        $this->processUser($user);
    }
});

// ✅ Chunk by ID (better for tables with deletions)
User::chunkById(1000, function ($users) {
    foreach ($users as $user) {
        $this->processUser($user);
    }
});

// ✅ Lazy collections (memory efficient)
User::lazy()->each(function ($user) {
    $this->processUser($user);
});

// ✅ Cursor (memory efficient for single loop)
foreach (User::cursor() as $user) {
    $this->processUser($user);
}
```

### Efficient Counting

```php
// ❌ Loads all records to count
$count = Post::all()->count();

// ✅ Database-level counting
$count = Post::count();

// ✅ Count with conditions
$publishedCount = Post::where('status', 'published')->count();

// ✅ Count relationships without loading
$users = User::withCount('posts')->get();
foreach ($users as $user) {
    echo $user->posts_count; // No additional query
}

// ✅ Multiple counts
$users = User::withCount(['posts', 'comments', 'likes'])->get();

// ✅ Conditional counts
$users = User::withCount([
    'posts' => fn ($query) => $query->where('published', true),
    'posts as draft_posts_count' => fn ($query) => $query->where('published', false),
])->get();
```

### Exists vs Count

```php
// ❌ Inefficient when you just need to know if records exist
if (Post::where('user_id', $userId)->count() > 0) {
    // Do something
}

// ✅ Use exists() - stops after first match
if (Post::where('user_id', $userId)->exists()) {
    // Do something
}

// ✅ Or doesntExist()
if (Post::where('user_id', $userId)->doesntExist()) {
    // Do something
}
```

---

## 5. Query Caching

### Basic Caching

```php
use Illuminate\Support\Facades\Cache;

// ✅ Cache query results
$users = Cache::remember('users.all', 3600, function () {
    return User::with('profile')->get();
});

// ✅ Cache with tags (Redis/Memcached only)
$posts = Cache::tags(['posts'])->remember('posts.recent', 3600, function () {
    return Post::with('author')
        ->where('published', true)
        ->latest()
        ->take(10)
        ->get();
});

// Clear tagged cache
Cache::tags(['posts'])->flush();
```

### Advanced Caching Strategies

```php
class PostRepository
{
    public function getPopularPosts(int $limit = 10): Collection
    {
        $cacheKey = "posts.popular.{$limit}";
        
        return Cache::remember($cacheKey, 3600, function () use ($limit) {
            return Post::select('posts.*')
                ->withCount('likes')
                ->orderBy('likes_count', 'desc')
                ->limit($limit)
                ->get();
        });
    }

    public function clearCache(): void
    {
        Cache::tags(['posts'])->flush();
    }

    public function getPost(int $id): ?Post
    {
        // Per-item caching
        return Cache::remember("post.{$id}", 3600, function () use ($id) {
            return Post::with(['author', 'tags', 'comments'])
                ->find($id);
        });
    }
}

// Observer to invalidate cache
class PostObserver
{
    public function saved(Post $post): void
    {
        Cache::forget("post.{$post->id}");
        Cache::tags(['posts'])->flush();
    }

    public function deleted(Post $post): void
    {
        Cache::forget("post.{$post->id}");
        Cache::tags(['posts'])->flush();
    }
}
```

---

## 6. Raw Queries & Subqueries

### When to Use Raw Queries

```php
// ✅ Complex aggregations
$stats = DB::table('orders')
    ->selectRaw('
        DATE(created_at) as date,
        COUNT(*) as total_orders,
        SUM(amount) as total_revenue,
        AVG(amount) as avg_order_value
    ')
    ->where('status', 'completed')
    ->groupBy('date')
    ->get();

// ✅ Subqueries
$latestPosts = Post::select('posts.*')
    ->selectSub(function ($query) {
        $query->selectRaw('COUNT(*)')
            ->from('comments')
            ->whereColumn('comments.post_id', 'posts.id');
    }, 'comments_count')
    ->having('comments_count', '>', 10)
    ->get();

// ✅ Join with subquery
$users = User::select('users.*')
    ->joinSub(
        Post::select('user_id')
            ->selectRaw('COUNT(*) as post_count')
            ->groupBy('user_id'),
        'post_stats',
        'users.id',
        '=',
        'post_stats.user_id'
    )
    ->where('post_stats.post_count', '>', 5)
    ->get();
```

---

## 7. Debugging & Monitoring Queries

### Query Logging

```php
// Enable query log
DB::enableQueryLog();

// Your queries here
$users = User::with('posts')->get();

// Get executed queries
$queries = DB::getQueryLog();
dd($queries);

// Custom query logger
DB::listen(function ($query) {
    Log::info('Query executed', [
        'sql' => $query->sql,
        'bindings' => $query->bindings,
        'time' => $query->time,
    ]);
});
```

### Slow Query Detection

```php
// AppServiceProvider
public function boot(): void
{
    DB::whenQueryingForLongerThan(500, function ($connection, $event) {
        Log::warning('Slow query detected', [
            'sql' => $event->sql,
            'bindings' => $event->bindings,
            'time' => $event->time,
            'connection' => $connection->getName(),
        ]);
    });
}
```

### Explain Query Plans

```php
// Get query explanation
$query = Post::with('author')
    ->where('published', true)
    ->orderBy('created_at', 'desc');

// View SQL
dd($query->toSql());

// View bindings
dd($query->getBindings());

// Explain query
DB::enableQueryLog();
$query->get();
$queries = DB::getQueryLog();

foreach ($queries as $query) {
    $explained = DB::select('EXPLAIN ' . $query['query'], $query['bindings']);
    dd($explained);
}
```

---

## 8. Optimization Best Practices

### Summary Checklist

```php
// ✅ DO:
✓ Use eager loading to prevent N+1 queries
✓ Select only required columns
✓ Add indexes to frequently queried columns
✓ Use chunk() or cursor() for large datasets
✓ Cache expensive queries
✓ Use exists() instead of count() > 0
✓ Use whereHas() with caution (can be slow)
✓ Monitor slow queries
✓ Use database transactions appropriately

// ❌ DON'T:
✗ Use all() for large tables
✗ Perform queries in loops
✗ Forget to index foreign keys
✗ Over-index (slows down writes)
✗ Use select * when you don't need all columns
✗ Ignore the explain output
✗ Cache data indefinitely
```

### Performance Comparison

```php
// Scenario: Get 1000 posts with authors and comments

// ❌ Worst: N+1 queries (2001 queries!)
$posts = Post::all();
foreach ($posts as $post) {
    echo $post->author->name;
    echo $post->comments->count();
}

// ⚠️ Better: Partial eager loading (1001 queries)
$posts = Post::with('author')->get();
foreach ($posts as $post) {
    echo $post->author->name;
    echo $post->comments->count(); // Still N+1 for comments
}

// ✅ Best: Full eager loading with count (3 queries)
$posts = Post::with('author')
    ->withCount('comments')
    ->get();
foreach ($posts as $post) {
    echo $post->author->name;
    echo $post->comments_count;
}

// 🚀 Optimal: Cached (0-3 queries depending on cache status)
$posts = Cache::remember('posts.with.stats', 3600, function () {
    return Post::select('id', 'title', 'user_id', 'created_at')
        ->with('author:id,name')
        ->withCount('comments')
        ->get();
});
```

---

## Conclusion

Query optimization is crucial for scalable Laravel applications. Start by identifying N+1 queries, add appropriate indexes, and implement caching where beneficial. Monitor your queries regularly and optimize as your application grows.

**Remember: Premature optimization is the root of all evil, but ignoring obvious performance issues is worse.**
