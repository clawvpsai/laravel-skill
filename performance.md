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

## Index Patterns — Composite Column Order Rules

A bare `index('column')` is fine for one-shot queries; production-grade schemas need **purpose-built composite indexes**. The rules below cover 95% of what you'll see in a slow-query log.

```php
use Illuminate\Database\Schema\Blueprint;

// 1. WHERE-only — single column
$table->index('author_id');

// 2. WHERE column_a = ? AND column_b = ?  — equality columns FIRST
$table->index(['author_id', 'status']);

// 3. WHERE a = ? ORDER BY b — a column for the WHERE, b second for the ORDER BY
$table->index(['author_id', 'created_at']);

// 4. WHERE a = ? AND b = ? ORDER BY c — all three columns, equality first
$table->index(['author_id', 'status', 'created_at']);

// 5. Covering index — include non-key columns via MySQL 8.0+ index "include"
//    Covers "WHERE author_id = ? — fetch me title + slug" without touching the row.
//    In Laravel 12+/13: $table->index(['author_id'], 'idx_posts_author_cover')
//                      ->include(['title', 'slug']);  // MySQL 8.0+ only

// 6. Partial / filtered index — only index rows that match a predicate
//    PostgreSQL:  $table->rawIndex('created_at', 'idx_active_posts',
//                                  'WHERE deleted_at IS NULL');
//    MySQL 8.0+:  no native partial index — use a generated column + index

// 7. Don't over-index
//    Every index slows INSERTs/UPDATEs (Postgres: HOT updates are blocked).
//    Indexes are read-amplification on writes.
```

**The composite-column rule (most-skipped rule in real codebases):**
- Equality columns first (`WHERE a = ? AND b = ?`)
- One range column last (`WHERE a = ? AND b > ?` — `(a, b)` works, `(b, a)` does not)
- ORDER BY columns at the end, matching the WHERE's equality/range order
- Example: `WHERE tenant_id = ? AND status = ? ORDER BY created_at DESC` → index `(tenant_id, status, created_at)`

**MySQL-only EXPLAIN signals worth watching:**
- `type: range` — better than `ALL` but worse than `ref`; composite usually improves it.
- `key_len` shorter than the column definition? Only a prefix is being used (e.g. `(10)` chars of `VARCHAR(255)`).
- `filtered: 100` but `rows: 1000`? Composite index isn't matching the way you think.

**PostgreSQL-only patterns:**
- `BRIN` for natural-order tables (logs, events) — 1000× smaller than btree, slower point lookups.
- `GIN` for full-text (`tsvector`) and `jsonb` containment (`@>`) queries.
- `gist` for geometric / range types.

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

## Cache Stampede Prevention — Lock Patterns (Laravel 12+ SWaR)

`Cache::remember()` alone is not enough — when a hot key expires, **all waiting workers compute the value in parallel**. That's the cache stampede / thundering herd. Two patterns solve it.

```php
use Illuminate\Support\Facades\Cache;

// === Method 1: Atomic lock — first worker computes, the rest wait ===
$stats = Cache::lock('stats.computation', 10)->block(5, function () {
    // Up to 5s wait. If we get the lock, recompute.
    return Cache::remember('stats', 3600, fn () => $this->computeStats());
});

// === Method 2: Stale-while-revalidate (built-in Laravel 12+ via Cache::flexible) ===
$stats = Cache::flexible('stats', [60, 600], function () {
    // First TTL = "fresh" window; second TTL = "stale but still served" window.
    // On hit within the stale window, the closure runs to recompute
    // WITHOUT invalidating the cached value first — readers keep getting
    // the old value while the recompute runs in the background.
    return $this->computeStats();
});
```

**Pattern selection:**
| Scenario | Pattern | Why |
|---|---|---|
| Key rarely expires, recompute is fast | `Cache::remember()` | Stampede is unlikely to matter |
| Key often expires, recompute is expensive | `Cache::lock()->block()` | One compute per expiration |
| Key often expires, recompute is expensive, **stale values are OK** | `Cache::flexible()` (SWaR) | Readers never block |
| Tagged keys, multi-key cache invalidation | `Cache::tags()` + lock pattern | Tags are Redis-only |

**Don't do these:**
```php
// WRONG — separate get/put has TOCTOU race; stampede-prone
if (Cache::has('stats')) {
    return Cache::get('stats');
}
$stats = $this->computeStats();   // 50 workers all do this on expiry
Cache::put('stats', $stats, 3600);
return $stats;

// WRONG — using the same lock key for everything → effectively serializes
$lock = Cache::lock('global-cache-lock');
```

**13.18.0 tagged fix (PR #60626):** `Cache::flexible()`'s internal `_flex_lock:` and `_flex_defer:` keys are now namespaced, so a user-provided `lockName` parameter can't collide with the internal defer key. Affects code that passes a custom `lockName` to `flexible()` — pre-13.18.0, a colliding custom name could silently fail to defer. Covered in `performance.md` TaggedCache section.

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
9. **Wrong composite-column order** — `(b, a)` instead of `(a, b)` makes index unusable for `WHERE a = ? AND b > ?` queries
10. **Cache stampede (thundering herd)** — `Cache::has()` + `Cache::get()` + `Cache::put()` lets many workers compute the same key in parallel
11. **Logging every DB::listen() query (no duration threshold)** — floods the log, hides real signal; threshold by `time` (default >250ms)

## `Number::forHumans` / `Number::abbreviate` / `Number::fileSize` OOM on `INF` / `NaN` (Laravel 13.18.0+)

Before 13.18.0, `Number::forHumans(INF)`, `Number::forHumans(-INF)`, `Number::forHumans(NAN)`, and `Number::abbreviate(INF)` all recursed into `Number::summarize()` without a base case for non-finite inputs. The process silently exhausted PHP memory and aborted. On PHP 8.5 there was also deprecation noise from `floor(log10(NAN))` -> int coercion. If your app renders a metric, a counter, a rate, or any user-controllable value through `Number::forHumans()` (e.g. a stats dashboard, a quota bar, a "1.2K views" label), an attacker can feed `INF` via a query string and crash the worker.

In 13.18.0 (PR #60617, commit 3a3c1d2b, 2026-06-27), the methods delegate to `Number::format()` and emit the locale-aware infinity, -infinity, NaN symbols. The matching fix to `Number::fileSize()` (PR #60625, commit 7e9d4aa1, 2026-06-29) ensures `Number::fileSize(INF)` no longer returns a unit-suffixed string for an unbounded value.

```php
use Illuminate\Support\Number;

// Pre-13.18.0: OOM-crashes the PHP process
Number::forHumans(INF);     // aborts (recursion, no base case)
Number::abbreviate(-INF);   // aborts
Number::forHumans(NAN);     // aborts + PHP 8.5 deprecation warning

// 13.18.0+: returns the locale-aware symbol
Number::forHumans(INF);     // "\u221e"
Number::forHumans(-INF);    // "-\u221e"
Number::forHumans(NAN);     // "NaN"
Number::abbreviate(INF);    // "\u221e"
Number::fileSize(INF);      // "\u221e" (no "B" / "KB" suffix)
```

**Why this is a performance bug, not just a UX bug:** the failure mode is a worker OOM, not a clean exception. A single request that lands on a code path rendering `Number::forHumans($someUserInput)` can take down a `php-fpm` pool or a queue worker. Treat it as a DoS vector: user input + an unbounded code path.

**Audit checklist:**
- `grep -r "Number::forHumans\|Number::abbreviate\|Number::fileSize" app/ resources/`
- For every hit, trace back: can the input be `INF` / `NaN`? A `count($hugeSet)` on a 64-bit PHP build cannot overflow to `INF` in normal usage, but `PHP_INT_MAX + 1` arithmetic, a division by zero in user input (`$users / $views`), or a malformed JSON payload can.
- If yes, either upgrade to 13.18.0+ or pre-validate the input to clamp to a finite value:

```php
// Defensive pre-check (works on all 13.x, no upgrade required)
function safeForHumans(float|int $n, int $max = PHP_INT_MAX): string {
    if (! is_finite($n)) {
        return Number::format(PHP_INT_MAX); // or a custom placeholder
    }
    return Number::forHumans(min(abs($n), $max));
}
```

**Related call sites worth hardening:**
- File-size display in admin panels (`Number::fileSize($upload->size)` is normally safe — `$size` is an int — but a value derived from `$request->input('bytes')` is not)
- Stats widgets that sum floats across a result set and then format with `Number::abbreviate`
- `count()` or `->count()` results from `Post::count() * 2.5` style ratios passed to `forHumans`

Source: [PR #60617 — Fix Number::forHumans and Number::abbreviate crashing on INF/NAN](https://github.com/laravel/framework/pull/60617) | [PR #60625 — Fix Number::fileSize wrong unit suffix for non-finite inputs](https://github.com/laravel/framework/pull/60625)

### `Number::forHumans()` / `Number::abbreviate()` Scaling Tiny Decimals (Laravel 13.20.0, PR #60768 by @daffa-aditya-p)

v13.20.0 patches a precision-loss bug: `Number::forHumans()` and `Number::abbreviate()` previously produced incorrect output for **very small fractional numbers** (e.g., `0.000001` -> malformed abbreviations, or wrong unit scale). The fix ensures scaling arithmetic is applied correctly before abbreviation.

```php
use Illuminate\Support\Number;

// Pre-13.20.0: tiny decimals produced wrong or empty output
Number::forHumans(0.000001);    // broken
Number::abbreviate(0.000001);   // broken

// v13.20.0+: correct micro/milli scaling
Number::forHumans(0.000001);    // "1 u" (micro) or locale-appropriate notation
Number::forHumans(0.001);      // "1 m" (milli) or locale-appropriate notation
Number::abbreviate(0.001);      // "0" (correctly rounds to nearest)
```

**When this matters:**
- Scientific or precision-tooling APIs displaying micro/milli unit notation
- Financial apps displaying very small fractions (basis points, micropayments)
- Any dashboard rendering normalized metrics that include sub-unit values

This is distinct from the 13.18.0 INF/NaN OOM fix (PR #60617 + #60625) — both patches are needed for complete `Number::` helper correctness.

Source: [PR #60768 — Fix Number::forHumans() and abbreviate() scaling tiny decimals](https://github.com/laravel/framework/pull/60768)

## Updated from Research (2026-07-10, cycle 32)

- **Added `## Index Patterns — Composite Column Order Rules` section** — covered the four-step slow-query diagnostic, the `equality → range → ORDER BY` rule for composite indexes, MySQL `EXPLAIN` red-flags (`type: ALL`, `Using filesort`, `key_len` mismatch), and PostgreSQL-specific patterns (`BRIN` for natural-order tables, `GIN` for full-text / `jsonb`, `gist` for geometric). Filament/Statamic/Spatie schemas commonly violate the composite-column rule silently — covered the most-skipped pattern in real codebases.
- **Added `## Cache Stampede Prevention — Lock Patterns (Laravel 12+ SWaR)` section** — proper coverage of `Cache::flexible()` stale-while-revalidate and `Cache::lock()->block()` first-computers-wait patterns, with the four-scenario pattern-selection matrix. Cross-referenced the PR #60626 tagged-key namespace fix (13.18.0+).
- **Updated Common Mistakes list** to grow 8 → 11 entries, adding "Cache stampede (thundering herd)", "Wrong composite-column order", and "Logging every DB::listen() query (no duration threshold)".

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
