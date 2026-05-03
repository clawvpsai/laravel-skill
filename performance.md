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

// Manual invalidation pattern
public function update(Post $post)
{
    Cache::forget('post.' . $post->id);
    Cache::forget('posts.active');
    // ...
}

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

## Pagination

```php
// Large tables — cursor pagination (faster)
Post::orderBy('id')->cursorPaginate(50);

// Standard (good for most cases)
Post::paginate(20);

// never do this for large datasets
Post::all(); // loads ALL records into memory
```

## Common Mistakes

1. **N+1 in loops** — always use `with()`
2. **Missing database indexes** — slow queries on large tables
3. **Counting in loops** — use `withCount()`
4. **Cache stampede** — use `Cache::remember()` not `Cache::get()` + `Cache::put()` separately
5. **No cache invalidation** — stale data served forever
6. **Loading too much in one query** — select only needed columns