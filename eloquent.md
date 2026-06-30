# Eloquent — Models, Relationships, Migrations

## The Golden Rules

### 1. Eager Load or Face N+1 Hell

**WRONG (N+1 — kills performance):**
```php
$posts = Post::all();
foreach ($posts as $post) {
    echo $post->author->name; // 1 query per post = 1000 queries
}
```

**RIGHT (eager load with with()):**
```php
$posts = Post::with('author')->get();
foreach ($posts as $post) {
    echo $post->author->name; // 2 queries total
}
```

**Prevent in dev:** Add to `AppServiceProvider::boot()`:
```php
if (app()->isLocal()) {
    Model::preventLazyLoading();
}
```

### 2. Use findOrFail() for Single Records

```php
// Returns null — silent failure
$post = Post::find($id);

// Throws 404 — explicit failure  
$post = Post::findOrFail($id);
```

### 3. $fillable vs $guarded — Always Whitelist

```php
// SECURE — only these fields can be mass-assigned
protected $fillable = ['title', 'body', 'author_id'];

// SECURE — all fields blocked except these explicitly set
protected $guarded = [];
```

**Never leave $guarded unset with user-submitted data.** That IS the mass assignment vulnerability.

## Migrations

```bash
# Create
php artisan make:migration create_posts_table

# Run
php artisan migrate

# Rollback
php artisan migrate:rollback

# Fresh (destroy + recreate)
php artisan migrate:fresh
```

**Schema builder patterns:**
```php
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->foreignId('author_id')->constrained()->cascadeOnDelete();
    $table->string('title');
    $table->text('body')->nullable();
    $table->timestamps();
    
    $table->index(['author_id', 'created_at']); // composite for common queries
});
```

**Never modify migrations after running on production.** Create a new one instead.

## Relationships

### One-to-Many
```php
// Model
public function posts()
{
    return $this->hasMany(Post::class);
}

// Access
$user->posts()->where('published', true)->get();
```

### Many-to-Many
```php
public function roles()
{
    return $this->belongsToMany(Role::class);
}
```

### Has-One-Through (skip-level)
```php
// Car → Mechanic → Owner (car has one owner through mechanic)
public function owner()
{
    return $this->hasOneThrough(Owner::class, Mechanic::class);
}
```

### Polymorphic
```php
// Imageable: Post and User can both have images
public function images()
{
    return $this->morphMany(Image::class, 'imageable');
}
```

## Query Builder

```php
// Chainable, safe from SQL injection
User::where('active', true)
    ->whereIn('role', ['admin', 'editor'])
    ->orderBy('created_at', 'desc')
    ->paginate(20);

// Aggregates
User::count();
User::where('active', true)->avg('posts_count');

// Subqueries
Post::where('created_at', now()->subDays(30))
    ->whereHas('author', fn($q) => $q->where('verified', true))
    ->with(['author', 'tags'])
    ->get();

// Nested where clauses (Laravel 12+)
$posts = Post::query()
    ->where(fn($q) => $q->where('status', 'published')
        ->orWhere('featured', true))
    ->where(fn($q) => $q->where('category', 'tech')
        ->orWhere('category', 'news'))
    ->get();
// Produces: WHERE (status = 'published' OR featured = 1) AND (category = 'tech' OR category = 'news')

// whereRelation (Laravel 10+)
$posts = Post::whereRelation('author', 'verified', true)->get();
$posts = Post::whereRelation('author', fn($q) => $q->where('role', 'admin'))->get();
```

**SortDirection Enum (Laravel 13.8+):**
```php
use Illuminate\Database\Query\SortDirection;

// Instead of string 'asc'/'desc', use type-safe SortDirection enum
Post::orderBy('created_at', SortDirection::Descending)->get();
Post::orderBy('title', SortDirection::Ascending)->get();

// Also works with collection sort/sortBy
$posts = $posts->sortBy('created_at', SortDirection::Descending);

// In query builder scope:
public function scopeOrderByDate($query, string $direction = 'desc')
{
    $sortDir = SortDirection::tryFrom($direction) ?? SortDirection::Descending;
    return $query->orderBy('created_at', $sortDir);
}
```

**Always use DB bindings — never string interpolation:**
```php
// SAFE
User::where('email', $email)->first();

// DANGEROUS — SQL injection vector
User::whereRaw("email = '$email'")->first();
```

## Accessors & Mutators

```php
// Accessor — $post->excerpt (auto-computed)
public function getExcerptAttribute(): string
{
    return Str::limit(strip_tags($this->body), 120);
}

// Mutator — auto-trim before saving
public function setTitleAttribute($value): void
{
    $this->attributes['title'] = trim($value);
}
```

## Observers

```php
// Option A: Observer class
php artisan make:observer PostObserver --model=Post

// Option B: Model boot method (for simple cases)
protected static function booted()
{
    static::created(fn(Post $post) => Log::info("Post {$post->id} created"));
}
```

## Soft Deletes

```php
// Model
use Illuminate\Database\Eloquent\SoftDeletes;

protected $fillable = ['title', 'body'];
protected $dates = ['deleted_at']; // auto on SoftDeletes

// Query
Post::withTrashed()->find($id); // include soft-deleted
Post::onlyTrashed()->get(); // soft-deleted only

// Restore / force delete
$post->restore();
$post->forceDelete();
```

## Scopes

```php
// Local scope — call as Post::published()->get()
public function scopePublished($query)
{
    return $query->where('published_at', '<=', now());
}

// Global scope (boot in model)
protected static function booted()
{
    static::addGlobalScope('active', fn($q) => $q->where('active', true));
}
```

## Factories (for testing/seeders)

```php
// Define
use Illuminate\Database\Eloquent\Factories\Factory;

class PostFactory extends Factory
{
    protected $model = Post::class;

    public function definition()
    {
        return [
            'title' => fake()->sentence(),
            'body' => fake()->paragraphs(3, true),
            'author_id' => User::factory(),
            'published_at' => fake()->optional()->dateTimeBetween('-1 year', 'now'),
        ];
    }

    public function unpublished() // variant
    {
        return $this->state(fn(array $attrs) => ['published_at' => null]);
    }
}

// Use
Post::factory()->count(50)->create();
Post::factory()->unpublished()->make();
```

## Common Mistakes

1. **Lazy loading in loops** — always check `with()`
2. **Using env() outside config** — config caches, env becomes null
3. **Mass assignment without $fillable** — exposes all fields
4. **Accessing deleted model** — always re-fetch after force delete
5. **N+1 with counts** — use `withCount('comments')` instead of `$post->comments->count()` in loop
6. **Not handling unique constraint violations** — wrap in try/catch or use `firstOrCreate`
7. **`nestedWhere` with complex AND/OR without parentheses** — always wrap OR groups in `where()` callback to ensure correct precedence

## Eloquent Strictness Hardening (`Model::preventX()`)

Laravel ships four global switches on `Illuminate\Database\Eloquent\Model` that turn silent, easy-to-miss bugs into loud failures. Every Laravel app should turn these on in **non-production** at minimum, and most should keep them on in production too. Highlighted at Laravel Live UK 2026 (June 29, 2026) as the single highest-leverage improvement teams can make to their Eloquent layer.

Recommended `AppServiceProvider::boot()` block:
```php
use Illuminate\Database\Eloquent\Model;

public function boot(): void
{
    Model::preventLazyLoading(! app()->isProduction());
    Model::preventSilentlyDiscardingAttributes(! app()->isProduction());
    Model::preventAccessingMissingAttributes(! app()->isProduction());
    Model::prohibitDestructiveCommands(
        ! app()->isProduction() && ! app()->runningInConsole()
    );
}
```

### 1. `Model::preventLazyLoading()` (Laravel 6+)

Throws `LazyLoadingViolationException` whenever you access a relation that wasn't eager-loaded.

```php
// Throws in non-prod: "Attempted to lazy load [author] on model [Post] but lazy loading is disabled."
foreach (Post::all() as $post) {
    echo $post->author->name;
}

// ✅ No exception
foreach (Post::with('author')->get() as $post) {
    echo $post->author->name;
}
```

**Use `Model::handleLazyLoadingViolationUsing()`** to override behavior — for example, log to Sentry in production instead of crashing:
```php
Model::handleLazyLoadingViolationUsing(function ($model, $key) {
    \Log::warning('lazy-load', ['model' => $model::class, 'key' => $key]);
});
```

### 2. `Model::preventSilentlyDiscardingAttributes()` (Laravel 9.28+)

Throws `MassAssignmentException` when `fill()` / `update()` is given a key that is **not** in `$fillable`. Catches the "I added a column to the DB but forgot to add it to `$fillable`" bug in dev — without this, the field is just silently dropped in production.

```php
class Post extends Model {
    protected $fillable = ['title', 'body']; // forgot 'slug'
}

// Throws in non-prod: "Add fillable property [slug] to allow mass assignment on [App\Models\Post]."
Post::create($request->all()); // request contains 'slug'
```

**This is the missing safety net between `$fillable` and runtime failures.** If `$fillable` was set correctly, this never fires. If it wasn't, you find out in dev instead of in production when the column mysteriously stays NULL.

### 3. `Model::preventAccessingMissingAttributes()` (Laravel 9.32+)

Throws `MissingAttributeException` when you access an attribute that was **never set on the instance** — either because it wasn't selected from the DB, isn't a column, or hasn't been `fill()`-ed.

```php
$user = User::select(['id', 'name'])->first(); // didn't select 'email'

// Throws in non-prod: "The attribute [email] either does not exist or was not retrieved for model [App\Models\User]."
echo $user->email;
```

**Catches two real bug classes:**
- **Typos in attribute access** — `$user->emial` instead of `$user->email` returns null normally; with this on, it throws.
- **Misconfigured `select()`** — you forgot to include a column the view template uses; the view shows empty strings instead of a clear error in dev.

### 4. `Model::prohibitDestructiveCommands()` (Laravel 11+)

Blocks `Model::delete()`, `Model::truncate()`, and any `forceDelete()` chain from running **outside of an `artisan` command**. Designed to prevent the classic "a web request or queue job accidentally wiped the table" disaster.

```php
// In a controller — BLOCKED in non-prod, throws MassAssignmentException-style error:
User::where('is_test', true)->delete();

// In php artisan tinker / migrate:fresh / db:seed — ALWAYS ALLOWED:
User::where('is_test', true)->delete(); // ✅ fine, artisan context
```

**Why this matters:** A bug like `$query->delete()` accidentally piped into a request handler can wipe the table in milliseconds. `prohibitDestructiveCommands` is the last-line-of-defense that turns that into a thrown exception instead of a production incident.

**Combine with `SoftDeletes`** — destructive ops should always be on `forceDelete()` paths, never on the default `delete()` path, so this catches the case where someone bypasses soft delete.

### Production-mode tuning

For production, you typically want **lazy loading to log but not crash** (so a missed `with()` doesn't 500 the user) but **silent discard and missing attribute to throw** (because those are bug-shaped, not performance-shaped):

```php
Model::preventLazyLoading(! app()->isProduction()); // throw in dev, allow in prod
Model::handleLazyLoadingViolationUsing(function ($model, $key) {
    // In prod, just log — don't crash a user-facing request for a perf issue
    \Sentry::captureMessage("lazy-load: {$model::class}->{$key}");
});

Model::preventSilentlyDiscardingAttributes(true); // ALWAYS on — this is a bug
Model::preventAccessingMissingAttributes(true);   // ALWAYS on — this is a typo
```

### Cross-cutting rule

Every Laravel project should have **all four** in `AppServiceProvider::boot()` in dev. The default Laravel skeleton ships with none — meaning every new app starts with the strictest defaults off. Add them on day one.

See [Laravel 13 Docs — Configuring Eloquent Strictness](https://laravel.com/docs/13.x/eloquent#configuring-eloquent-strictness).

</content>
</invoke>## Postgres Transaction Pooler Support (Laravel 13.17+)

When you run Laravel behind a Postgres connection pooler in **transaction mode** (PgBouncer, Neon, Supabase pooler, AWS RDS Proxy in transaction mode), persistent connections are forbidden — every transaction must run on a freshly checked-out connection. Laravel 13.17 (PR #60425) added first-class support for this in `Illuminate\Database\PostgresConnection`:

```php
// config/database.php — connection string for transaction-pooler platforms
'pgsql' => [
    'driver' => 'pgsql',
    'host'   => env('DB_HOST'),           // pooler hostname (e.g. ep-xxx.pooler.supabase.com)
    'port'   => env('DB_PORT', 6543),     // pooler port (6543 for Supabase, 6432 for PgBouncer)
    'database' => env('DB_DATABASE'),
    'username' => env('DB_USERNAME'),
    'password' => env('DB_PASSWORD'),
    'sslmode' => 'require',
    // PostgresConnection now resets connection-scoped state (search_path, SET LOCAL,
    // advisory locks, temp tables) between transactions automatically.
],
```

**Why this matters:** without the fix, `SET LOCAL` from one transaction would leak into the next, advisory locks would be held by the wrong session, and prepared statements would target the wrong schema — all silently. With 13.17+, `PostgresConnection::disconnect()` clears the state before returning the connection to the pool.

**Pair with the new disconnect fix:** PR #60574 (same release) clears the transaction manager state on DB disconnect — so if your pooler cuts a connection mid-transaction, the framework no longer leaves a phantom savepoint in the new connection that would later raise "current transaction is aborted, commands ignored until end of transaction block".

**Things that still don't work in transaction mode:**
- `BEGIN; ... COMMIT;` from raw SQL containing `PREPARE` / `LISTEN`
- Long-running advisory locks held across multiple HTTP requests (use cache locks instead)
- Session-scoped `SET` statements (use `SET LOCAL` inside the transaction)

See `deployment.md` (Postgres Transaction Pooler Support section) for connection-string examples for Neon / Supabase / RDS Proxy.

## Transaction State Cleared on Disconnect (Laravel 13.17+)

Before 13.17, a DB disconnect mid-transaction would leave the transaction manager believing an open transaction still existed. The next query on the new connection would fail with the cryptic "current transaction is aborted, commands ignored until end of transaction block" Postgres error — even though the new connection had no transaction at all.

PR #60574 fixes this by resetting the transaction manager when a connection is lost and re-established:

- Phantom savepoints no longer survive a failover
- Auto-commit returns immediately after a `Connection::disconnect()` cycle
- Retry helpers (e.g. `DB::transaction($cb, $attempts)`) recover cleanly on transient disconnects

If you see that error in logs after a Postgres failover or pooler rotation, upgrade to 13.17+.

## Updated from Research (2026-06-30, cycle 13)

- **Eloquent Strictness Hardening (`Model::preventX()`)** — added a new "Eloquent Strictness Hardening" section consolidating all four `Model::preventX()` switches (`preventLazyLoading`, `preventSilentlyDiscardingAttributes`, `preventAccessingMissingAttributes`, `prohibitDestructiveCommands`). Previously only `preventLazyLoading` was mentioned (line 26). The other three — promoted heavily at Laravel Live UK 2026 on June 29, 2026 — were the highest-leverage gap: they convert silent Eloquent bugs (typo'd attribute, missing `$fillable`, accidental mass-delete) into loud exceptions in dev. Recommended `AppServiceProvider::boot()` block + production-mode tuning (lazy-load log-to-Sentry, but throw on silent-discard/missing-attribute) included.

---

## Updated from Research (2026-06-26, cycle 5)

- **Postgres Transaction Pooler Support (Laravel 13.17+)** — PR #60425 by @DGarbs51 adds first-class support for PgBouncer / Neon / Supabase / RDS Proxy in transaction mode. `PostgresConnection` correctly resets connection-scoped state (`SET LOCAL`, advisory locks, temp tables, `search_path`) between transactions. Pair with the disconnect fix (PR #60574) below. See the detailed section above.
- **Transaction State Cleared on Disconnect (Laravel 13.17+)** — PR #60574 resets the transaction manager when a connection is lost, eliminating the "current transaction is aborted, commands ignored until end of transaction block" error after a Postgres failover or pooler rotation. See the detailed section above.

---

## Updated from Research (2026-05-18)

- **SortDirection enum (Laravel 13.8+)** — `Illuminate\Database\Query\SortDirection` provides type-safe `Ascending`/`Descending` values for `orderBy()` instead of string `'asc'`/`'desc'`
- **nestedWhere()** (Laravel 12+) — cleaner alternative to deeply nested closures for mixed AND/OR conditions
- **whereRelation()** (Laravel 10+) — `whereRelation('author', 'verified', true)` reads more naturally than `whereHas`


Source: [Laravel 13 Docs - Eloquent](https://laravel.com/docs/13.x/eloquent) | [Laravel 12 Query Builder](https://laravel.com/docs/12.x/queries)
