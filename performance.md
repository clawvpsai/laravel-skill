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

## Semantic Search — pgvector (Laravel 13 AI Search)

Laravel 13 supports **semantic/vector search** using PostgreSQL's pgvector extension. Instead of matching exact keywords, it matches by *meaning*. Requires:
- PostgreSQL with `pgvector` extension enabled
- `illuminate/db` PostgreSQL driver

**Migration — create vector column:**
```php
Schema::create('knowledge_articles', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->text('content');
    $table->json('embedding'); // stores the vector (typically 1536 dims for OpenAI embeddings)
    $table->timestamps();
});

// Enable pgvector
DB::statement('CREATE EXTENSION IF NOT EXISTS vector');
```

**Model with vector column:**
```php
use Illuminate\Database\Eloquent\Model;

class KnowledgeArticle extends Model
{
    protected $casts = [
        'embedding' => 'array', // casts vector JSON to PHP array
    ];
}
```

**Generate and store embeddings (Laravel AI SDK):**
```php
use Laravel\AI\Facades\Embeddings;

// Generate embedding for content
$embedding = Embeddings::embed($text); // returns float[]

// Store with article
$article = KnowledgeArticle::create([
    'title' => $title,
    'content' => $text,
    'embedding' => $embedding,
]);
```

**Query by semantic similarity:**
```php
// Find articles semantically similar to a query
$queryEmbedding = Embeddings::embed($searchQuery);

$results = KnowledgeArticle::query()
    ->whereVectorSimilarTo('embedding', $queryEmbedding)
    ->orderByRelevance('embedding', $queryEmbedding) // orders by most similar
    ->limit(5)
    ->get();
```

**RAG (Retrieval Augmented Generation) pattern:**
```php
// 1. Retrieve relevant context via semantic search
$context = KnowledgeArticle::query()
    ->whereVectorSimilarTo('embedding', $userQueryEmbedding)
    ->limit(3)
    ->get()
    ->pluck('content')
    ->join("\n\n");

// 2. Feed context to LLM
$response = AI::prompt('openai', 'gpt-4o', [
    'model' => 'gpt-4',
    'messages' => [
        ['role' => 'system', 'content' => "Use this context: {$context}"],
        ['role' => 'user', 'content' => $userQuery],
    ],
]);
```

**When to use semantic search vs traditional:**
- **Semantic search** — user queries with natural language ("how do I reset my password?"), documents with similar *meaning* are found even without exact keyword overlap
- **Traditional `where()`/`fulltext()`** — exact term matching, structured filters, known categories
- **Hybrid** — run both, merge/rank results

## MariaDB Vector Index (Laravel 13.13+)

Laravel 13.13 adds first-class support for **MariaDB vector indexes**, enabling vector/embedding search on MariaDB (instead of just PostgreSQL/pgvector). Requires MariaDB with vector support enabled.

**Migration — create vector column + index:**
```php
use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;

// Enable vector extension (run once per database)
DB::statement('CREATE EXTENSION IF NOT EXISTS vector');

// Migration
Schema::create('knowledge_articles', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->text('content');
    $table->vectorIndex(['embedding'], 'vector_index', 'vector_length:1536');
    $table->timestamps();
});
```

**Key parameters:**
- `['embedding']` — column(s) to index
- `'vector_index'` — index name
- `'vector_length:1536'` — embedding dimensions (1536 for OpenAI text-embedding-3-small, 3072 for large)

**Query by similarity (same as pgvector):**
```php
$queryEmbedding = Embeddings::embed($searchQuery);

$results = KnowledgeArticle::query()
    ->whereVectorSimilarTo('embedding', $queryEmbedding)
    ->orderByRelevance('embedding', $queryEmbedding)
    ->limit(5)
    ->get();
```

**MariaDB vs PostgreSQL for vectors:**
| | MariaDB | PostgreSQL |
|---|---|---|
| Package | `laravel/ai` + MariaDB | `laravel/ai` + pgvector |
| Index type | `VECTOR` | `vector` extension |
| Query syntax | Same `whereVectorSimilarTo()` | Same |
| Production readiness | Newer (Laravel 13.13) | Mature |

**When to use MariaDB over PostgreSQL:**
- Already running MariaDB (no second database needed)
- Simpler stack — one DB for everything
- MariaDB 11.x+ with vector support enabled


## Common Mistakes

1. **N+1 in loops** — always use `with()`
2. **Missing database indexes** — slow queries on large tables
3. **Counting in loops** — use `withCount()`
4. **Cache stampede** — use `Cache::remember()` not `Cache::get()` + `Cache::put()` separately
5. **No cache invalidation** — stale data served forever
6. **Loading too much in one query** — select only needed columns
7. **No transactions for related writes** — partial updates on failure
8. **Running heavy work synchronously** — block users, timeout issues

## Updated from Research (2026-06-26, cycle 5)

- **Cache Debounce `maxWait` Performance Fix (Laravel 13.17+)** — PR #60559 by @jackbayliss fixes `Cache::flexible()` / `Cache::remember()` with `->debounce(...)->maxWait(...)` so the underlying source is not re-checked on every call inside the debounce window. Pre-13.17, hot idempotency keys and rate limit counters could trigger 10–100× the expected closure / DB calls. See the detailed section below.

---

## Updated from Research (2026-06-19)

### Semantic Search (pgvector) — Laravel 13

- **Vector search via `whereVectorSimilarTo()`** — semantic search using pgvector. Matches results by meaning, not keywords.
- **RAG pattern** — retrieve relevant documents via vector search, feed as context to LLM for AI-powered answers.
- **Embeddings facade** — `Laravel\AI\Facades\Embeddings::embed()` generates vector embeddings for content.
- Use when users search with natural language queries where exact keyword matching falls short.
- Consider hybrid: run traditional `where()` + semantic search, merge ranked results.

Sources: [Laravel 13 Docs - Search](https://laravel.com/docs/13.x/search) | [Laravel News - RAG with pgvector](https://laravel-news.com/ship-ai-with-laravel-rag-with-embeddings-and-pgvector-in-laravel-13)

### Performance Patterns

- **Cursor pagination** — memory-efficient iteration over large datasets
- **Cache stampede prevention** — use lock keys to prevent thundering herd
- **Pessimistic locking** — `lockForUpdate()` for race-condition-safe updates
- **Conditional eager loading** — `with(['relation' => fn()])` to filter loaded relations
- **withCount with conditions** — count with specific filters via alias

Source: [Laravel 13 Docs - Eloquent Relationships](https://laravel.com/docs/13.x/eloquent)

### Cache::touch() — Extend TTL Without Re-Fetching (Laravel 13)

`Cache::touch()` updates the expiration time of a cached item **without** re-fetching or modifying its value. Useful for sliding expiration patterns (sessions, activity tracking):

```php
use Illuminate\Support\Facades\Cache;

// Extend TTL by 1 hour — value stays the same
Cache::touch('user.last_active.' . $user->id, 3600);

// Sliding session cache — accessed every time, expires 30 min after last access
Cache::put('session.' . $sessionId, $data, 1800);
Cache::touch('session.' . $sessionId, 1800); // reset TTL on each activity
```

**Use cases:**
- User "last seen" tracking — update TTL on each page view without re-writing data
- Sliding window caches — frequently accessed data stays hot, rarely used items expire
- Session-like patterns where access = keep-alive

**Caveats:**
- `touch()` only works on cache backends that support it natively (Redis, Memcached, Array). File/DB backends may not support it — check driver capabilities.
- Returns `bool` — `false` if key doesn't exist or backend doesn't support it.

Source: [Laravel 13 Docs - Cache](https://laravel.com/docs/13.x/cache)

### Cache::rememberWithWarmth() — Know If the Cache Was Hit (Laravel 13.15)

`Cache::rememberWithWarmth()` is the stateful twin of `Cache::remember()`. Same TTL/closure semantics, but it returns a tuple of `[$value, $wasWarm]` so you can tell whether the value came from the cache or was just computed. Useful for instrumentation, response headers, debug logging, or avoiding extra work downstream when the result is fresh.

```php
use Illuminate\Support\Facades\Cache;

// [$value, $wasWarm] — $wasWarm is true if it was already cached
[$stats, $wasCached] = Cache::rememberWithWarmth('site.stats', 3600, fn() => $this->computeStats());

// Surface cache-hit info in HTTP responses
return response()
    ->json(['stats' => $stats])
    ->header('X-Cache', $wasCached ? 'HIT' : 'MISS');

// Skip an expensive post-process step when the cached value is already known good
[$report, $wasCached] = Cache::rememberWithWarmth('weekly-report', 86400, fn() => $this->buildReport());
if (! $wasCached) {
    AuditLog::record('report.regenerated', ['user_id' => auth()->id()]);
}
```

**Why it exists:** The original PR author (cosmastech) wanted to record cache-hit/miss in trace logs and HTTP headers without an extra `Cache::has()` round trip. The first PR title called it `rememberWithState()` but the author noted that name "is dogshit" and the merged name is `rememberWithWarmth()`.

**Notes:**
- Return type is `array{TCacheValue, bool}` — first element is the value, second is `true` for cache hit / `false` for cache miss.
- `Cache::remember()` is now a thin wrapper that returns `rememberWithWarmth(...)[0]`, so existing call sites are unaffected.
- Pairs well with `Cache::flexible()` — call `Cache::flexible($key, [5, 60], fn() => $data)` then wrap with `rememberWithWarmth()` if you also need warm/cold state.

Source: [PR #60385 — Cache `rememberWithWarmth()`](https://github.com/laravel/framework/pull/60385)

### Cache Debounce `maxWait` Performance Fix (Laravel 13.17+)

Before 13.17, calling `Cache::flexible()` / `Cache::remember()` with `->debounce(...)->maxWait(...)` would re-evaluate the underlying closure on every call *even while inside the debounce window*, because each call independently checked whether the debounce had elapsed. On hot idempotency keys (rate limiters, payment idempotency tokens, request dedupe) this produced 10–100× the expected closure calls and DB hits.

In 13.17 (PR #60559 by @jackbayliss), once a debounce window is armed the underlying source is not re-checked until the window actually expires — matching the documented intent of `debounce + maxWait`:

```php
// Pre-13.17: closure called on every call within the debounce window
// 13.17: closure called only when debounce window expires (or maxWait triggers)
$value = Cache::flexible($key, [5, 10], fn() => $this->expensiveFetch())
    ->debounce(2)         // batch duplicate requests within 2s
    ->maxWait(60);        // but always recompute at least once per 60s
```

**When this matters:**
- Idempotency keys for "did this webhook already arrive?" lookups under retry storms
- Rate limit counters using cache-backed sliding windows
- Background-job dedupe keys hit from many web workers
- `Cache::remember()` for short-lived tokens (CSRF nonce, password-reset codes)

**How to verify the fix is active:** add a temporary counter inside the closure and burst-fire 100 requests for the same key within 2 seconds — on 13.17+ the counter increments once (or at most once per `maxWait`); on older versions it increments ~100×.

**Audit checklist:** if you have `->debounce()` chains in production and were seeing DB / cache backend load that did not match your traffic shape, this is the upgrade that fixes it.

See `queues.md` (Debounceable Jobs section) for the parallel queue-job debounce pattern.

Source: [PR #60559 — Reduce cache hits when debouncing with maxWait](https://github.com/laravel/framework/pull/60559) | [Laravel 13.17 Release Notes](https://github.com/laravel/framework/releases/tag/v13.17.0)
