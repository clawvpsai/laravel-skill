# Performance — Caching, Indexes, Eager Loading

## N+1 Query Prevention

```php
// ALWAYS eager load relationships when looping
// WRONG — N+1 query problem
$posts = Post::all();
foreach ($posts as $post) {
    echo $post->author->name; // 1 extra query per post = 1001 queries
}

// RIGHT — 2 queries total
$posts = Post::with('author')->get();
foreach ($posts as $post) {
    echo $post->author->name;
}

// Multiple relationships
$posts = Post::with(['author', 'comments', 'tags'])->get();

// Nested eager loading
$posts = Post::with('author.profile')->get();

// Conditional eager loading
$posts = Post::with(['author' => fn($q) => $q->select('id', 'name')])->get();
```

## Lazy Loading Prevention (Dev Only)

```php
// AppServiceProvider boot() — crashes app on N+1 in development
if (app()->isLocal()) {
    Model::preventLazyLoading();
}
```

## Indexing

```php
// Migrations — add indexes for frequently queried columns
Schema::table('posts', function (Blueprint $table) {
    $table->index('author_id');
    $table->index(['author_id', 'created_at']); // composite
    $table->unique(['slug']); // unique constraint
});
```

**When to index:**
- Foreign keys (`user_id`, `author_id`)
- Columns used in `where()` clauses frequently
- Columns used in `orderBy()`
- Composite indexes for multi-column filters

## Counting in Loops

```php
// WRONG — extra query per post
foreach ($posts as $post) {
    echo $post->comments->count(); // N+1
}

// RIGHT — withCount
$posts = Post::withCount('comments')->get();
foreach ($posts as $post) {
    echo $post->comments_count;
}

// With nested counts
$posts = Post::withCount(['comments', 'comments.votes'])->get();

// With specific conditions
$posts = Post::withCount([
    'comments as approved_comments_count' => fn($q) => $q->where('approved', true)
])->get();
```

## Caching

```php
use Illuminate\Support\Facades\Cache;

// Cache query results
$posts = Cache::remember('posts.active', 3600, fn() => 
    Post::with('author')->where('active', true)->get()
);

// Invalidate on update
Cache::forget('posts.active');
Cache::put('posts.active', $posts, 3600);

// Cache tags (Redis)
Cache::tags(['posts'])->put('active', $posts, 3600);
Cache::tags(['posts'])->flush(); // clear all posts cache
```

## Cache Layers

```php
// Fast check → expensive operation
if (Cache::has('stats')) {
    $stats = Cache::get('stats');
} else {
    $stats = $this->computeStats();
    Cache::put('stats', $stats, 3600);
}

// Or remember (lazy)
$stats = Cache::remember('stats', 3600, fn() => $this->computeStats());

// Cache stampede prevention — lock key
$stats = Cache::remember('stats', 3600, function () {
    // Only one process computes at a time
    return Cache::remember('stats.lock', 5, fn() => false) ?: $this->computeStats();
});
```

## Database Transactions

```php
// Wrap related writes in transactions for performance + consistency
DB::transaction(function () use ($post, $user) {
    $post->update(['author_id' => $user->id]);
    $user->increment('post_count');
    $this->indexPost($post->id);
});

// For long-running transactions, use pessimistic locking
DB::transaction(function () {
    $post = Post::lockForUpdate()->find($postId);
    // Row is locked until transaction commits
    $post->update(['status' => 'published']);
});
```

## Query Optimization

```php
// Select only needed columns
$posts = Post::select('id', 'title', 'author_id')->with('author:id,name')->get();

// Chunk large datasets instead of all()
Post::where('active', true)->chunk(100, function ($posts) {
    foreach ($posts as $post) {
        // Process each batch
    }
});

// Cursor (memory efficient for very large datasets)
foreach (Post::where('active', true)->cursor() as $post) {
    // One query, low memory
}

// Avoid counts in loops — use withCount or separate query
```

## Pagination

```php
// Large tables — cursor pagination (faster)
Post::orderBy('id')->cursorPaginate(50);

// Standard (good for most cases)
Post::paginate(20);

// never do this for large datasets
Post::all(); // loads ALL records into memory
```

## Queue Workers for Heavy Work

```php
// Offload heavy computation to queue instead of HTTP request
// Controller
ProcessPostJob::dispatch($postId)->onQueue('processing');

// Job (runs async, doesn't block user request)
class ProcessPostJob implements ShouldQueue
{
    public function handle(): void
    {
        // Heavy computation here
        $this->generateReport();
        $this->updateSearchIndex();
    }
}
```

## Query Logging (Dev)

```php
// DB::listen() in AppServiceProvider
use Illuminate\Support\Facades\DB;

if (app()->isLocal()) {
    DB::listen(function ($query) {
        logger()->info($query->sql, [
            'bindings' => $query->bindings,
            'time' => $query->time,
        ]);
    });
}
```

## Common Mistakes

1. **N+1 in loops** — always use `with()`
2. **Missing database indexes** — slow queries on large tables
3. **Counting in loops** — use `withCount()`
4. **Cache stampede** — use `Cache::remember()` not `Cache::get()` + `Cache::put()` separately
5. **No cache invalidation** — stale data served forever
6. **Loading too much in one query** — select only needed columns
7. **No transactions for related writes** — partial updates on failure
8. **Running heavy work synchronously** — block users, timeout issues

## Updated from Research (2026-05)

- **Cursor pagination** — memory-efficient iteration over large datasets
- **Cache stampede prevention** — use lock keys to prevent thundering herd
- **Pessimistic locking** — `lockForUpdate()` for race-condition-safe updates
- **Conditional eager loading** — `with(['relation' => fn()])` to filter loaded relations
- **withCount with conditions** — count with specific filters via alias

Source: [Laravel 13 Docs - Eloquent Relationships](https://laravel.com/docs/13.x/eloquent)