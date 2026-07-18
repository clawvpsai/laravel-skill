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

**Multi-column `whereAll()` / `whereAny()` (Laravel 10.47+):**
```php
// whereAny — match if AT LEAST ONE column contains the value (OR across columns)
User::query()
    ->whereAny(['first_name', 'last_name', 'email'], 'LIKE', "%{$search}%")
    ->get();
// SQL: WHERE (first_name LIKE ? OR last_name LIKE ? OR email LIKE ?)

// whereAll — match if ALL columns contain the value (AND across columns)
User::query()
    ->whereAll(['first_name', 'last_name'], 'LIKE', "{$term}%")
    ->get();
// SQL: WHERE (first_name LIKE ? AND last_name LIKE ?)

// orWhereAny / orWhereAll — same logic but as OR-joined to the previous group
User::query()
    ->where('active', true)
    ->orWhereAny(['first_name', 'last_name'], 'LIKE', "%{$search}%")
    ->get();
```

**When to pick `whereAny` over `orWhere(fn)`:** Use `whereAny` when you have a flat list of columns and a single value/operator — it generates cleaner SQL and is faster (no closure overhead). Use `orWhere(fn($q) => ...)` when you need different operators or values per column, or when you need to wrap complex sub-clauses.

**`whereRelation` / `whereMorphRelation` (Laravel 10+/12+):**
```php
// whereRelation — relationship-scoped whereHas with shorter syntax
$posts = Post::whereRelation('author', 'verified', true)->get();
$posts = Post::whereRelation('author', fn($q) => $q->where('role', 'admin'))->get();

// whereRelation with operator
$posts = Post::whereRelation('author', 'followers', '>', 1000)->get();

// whereMorphRelation (Laravel 12+) — same for polymorphic relations
$activities = Activity::whereMorphRelation('parentable', [Post::class, Video::class], 'featured', true)->get();
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
// Classic local scope — call as Post::published()->get()
public function scopePublished($query)
{
    return $query->where('published_at', '<=', now());
}

// Laravel 12+ — typed #[Scope] attribute (no `scopeXxx` naming convention required)
use Illuminate\Database\Eloquent\Attributes\Scope;
use Illuminate\Database\Eloquent\Builder;

#[Scope]
protected function published(Builder $query): void
{
    $query->where('published_at', '<=', now());
}

// Dynamic scope (with parameters) — also works as an attribute
#[Scope]
protected function ofType(Builder $query, string $type): void
{
    $query->where('type', $type);
}

// Global scope (boot in model)
protected static function booted()
{
    static::addGlobalScope('active', fn($q) => $q->where('active', true));
}
```

**`#[Scope]` attribute (Laravel 12+):**
- Replaces the `scopeXxx()` method-name convention. Method can be `protected` (preferred) or `public`.
- **Must be `protected`** per Laravel docs — `public` works but the docs flag it as discouraged.
- Calling an attributed scope from inside the model class: `static::query()->ofType('admin')` — must go through the query builder, not `$this->ofType()` directly (the latter doesn't route through Eloquent's scope handling).
- Refactor-rename safe (no magic prefix to keep in sync) and IDE-discoverable (the attribute shows up in `Find Usages` / autocompletion where a string-prefixed method name wouldn't).
- Dynamic scopes work the same way: extra method parameters become scope arguments.
- Both styles (`scopeXxx()` and `#[Scope]`) coexist. New code should prefer the attribute.

**Common gotcha:** the attribute only marks the method as a scope. The method signature still needs `Builder $query` as the first parameter — the framework dispatches via reflection but the SQL builder is still injected positionally.

Source: [Laravel 13 Docs - Eloquent Query Scopes](https://laravel.com/docs/13.x/eloquent#query-scopes) | [PHP Attributes in Laravel 13 Guide](https://laraveldaily.com/post/php-attributes-in-laravel-13-the-ultimate-guide-36-new-attributes)

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


## Collection Aggregation — `reduceInto()` and `reduceMany()` (Laravel 13.15+ / 13.19+)

Eloquent `Collection` and `Support\Collection` ship two specialised aggregation helpers that complement the existing `reduce()` and `sum()` / `groupBy()` / `pluck()`:

```php
// reduceMany(fn($c, $v) => ..., $init) — same shape as reduce() but skips the
// empty-collection seed branch for a small speedup. Shipped 13.15 (PR #60600 ish).
$total = $orders->reduceMany(fn($carry, $order) => $carry + $order->amount, 0);

// reduceInto(fn($c, $v) => ..., $init) — 13.19+ (PR #60651 by @JosephSilber).
// Same speedup but explicitly named to communicate "we want the accumulated result,
// not the per-step carry chain." Pairs with Arr::reduceInto() (also 13.19+).
$maxScore = $scores->reduceInto(
    fn($carry, $score) => max($carry, $score),
    PHP_INT_MIN,
);

// 13.19+ also adds: collection->counted() — character-aware count (delegates to Countable rules)
$orders->counted(); // count($orders), but on Stringable / Eloquent Collection the same as count()
```

**When to use which:**

| Method | Returns | When to use |
|---|---|---|
| `->sum('amount')` | numeric | one-shot numeric aggregation, indexed access |
| `->pluck('amount')->sum()` | numeric | aggregation across a plucked column |
| `->reduce(fn, $init)` | mixed | general-purpose fold; the `init` is the seed AND final value |
| `->reduceMany(fn, $init)` | mixed | 13.15+, faster when you don't read `$carry` between iterations |
| `->reduceInto(fn, $init)` | mixed | 13.19+, fastest of the three; semantic intent is "result not carry" |

**Performance note:** `reduceMany` and `reduceInto` are micro-optimisations — typically 10–20% faster than `reduce()` on large collections because they skip the empty-initial seeding branch. Only matters on hot paths (10k+ items). For most use cases, the readability of `reduce()` is worth the marginal speed loss. PR #60651 by @JosephSilber.

```php
// Real-world use case: building a histogram from a collection
$ordersByStatus = $orders->reduceInto(
    fn(array $hist, Order $order) => array_merge($hist, [
        $order->status->value => ($hist[$order->status->value] ?? 0) + 1,
    ]),
    [],
);
// ['pending' => 12, 'shipped' => 47, 'delivered' => 128]
```

## PostgreSQL `whereDate` / `whereTime` Expression Fix (Laravel 13.19+ / 12.63+)

Before 13.19 (and 12.63), `whereDate(DB::raw('foo'), '>=', '...')` crashed on PostgreSQL with `column "foo" does not exist` because the column-quoting logic treated the `Expression` as a raw identifier and tried to quote it. 13.19+ unwraps the expression and emits the raw SQL:

```php
// 13.19+: works on PostgreSQL — before, this threw "column foo does not exist"
$rows = DB::table('orders')
    ->whereDate(DB::raw('created_at::date'), '>=', now()->subWeek())
    ->get();

// Same fix for whereTime
$rows = DB::table('orders')
    ->whereTime(DB::raw('extract(hour from created_at)'), '>=', 9)
    ->get();
```

**Why this matters:** PostgreSQL apps that use raw expressions in date queries (e.g. `created_at::date` for date truncation, `extract(hour from created_at)` for hour-level filtering) had to switch to `whereRaw()` as a workaround. Now they can use the typed `whereDate()` / `whereTime()` builders and get the benefit of parameter binding without the column-quoting bug. PR #60540 (backported to 12.63.0 as well).

## Common Mistakes

## Quiet Bulk Increment / Decrement — `incrementEachQuietly()` / `decrementEachQuietly()` (Laravel 13.20+, PR #60720)

Firing hundreds of individual `updated` model events for a simple view-count update is expensive. Laravel 13.20 adds the **quiet** variants that bypass per-record model events entirely, firing a single `updated` for the whole query:

```php
// Before — fires N individual model events (one per record)
foreach ($products as $product) {
    $product->increment('views');   // updated + incrementing + incremented events × N
}

// Laravel 13.20 — fires ONE query, ONE updated event total
Product::whereIn('id', $ids)->incrementEachQuietly(['views' => 1]);

// Multi-column decrement (e.g., stock management)
OrderItem::where('status', 'shipped')
    ->decrementEachQuietly(['quantity' => 1, 'reserved' => 1]);

// Mixed increment and decrement
Metric::where('date', today())->incrementEachQuietly([
    'visits' => 1,
    'bounces' => -1,
]);
```

**What fires:** one `updated` event on the parent model (not N individual model events). `incrementing` / `decrementing` / `incremented` / `decremented` events are **not** fired. Use for analytics counters, view counts, stock levels, and any bulk metric that doesn't need per-record event propagation.

**When to use plain `increment()` vs `incrementEachQuietly()`:**
| Scenario | Method |
|---|---|
| Single record counter | `Model::increment('counter')` |
| Bulk analytics/counters, no observers needed | `Model::query()->incrementEachQuietly([...])` |
| You need `updated` model events per record | Use a chunked loop with `increment()` |
| Inventory/stock — observers must react | Chunked loop with `increment()` + observer |


1. **Lazy loading in loops** — always check `with()`
2. **Using env() outside config** — config caches, env becomes null
3. **Mass assignment without $fillable** — exposes all fields
4. **Accessing deleted model** — always re-fetch after force delete
5. **N+1 with counts** — use `withCount('comments')` instead of `$post->comments->count()` in loop
6. **Not handling unique constraint violations** — wrap in try/catch or use `firstOrCreate`
7. **`nestedWhere` with complex AND/OR without parentheses** — always wrap OR groups in `where()` callback to ensure correct precedence
8. **Forgetting `embedding` cast on `vector` columns** — without `'embedding' => 'array'` cast, the attribute comes back as a raw `pgvector` literal string (e.g. `'[0.05,0.10,...]'`) and your code has to parse it by hand. Always cast `vector` columns.
9. **Using `whereVectorSimilarTo` without `pgvector` enabled** — the query fails with `extension "vector" is not available`. Run `CREATE EXTENSION IF NOT EXISTS vector;` in your database first, or use `Schema::ensureVectorExtensionExists()` in a migration.
10. **Eager-loading the world with `Model::automaticallyEagerLoadRelationships()`** — the feature only fires when a relation is *accessed* on a model inside a collection, and only loads what was touched. It does NOT override an explicit `->select(['id', 'name'])` — you still get the N+1 on missing columns. Combine with `Model::preventAccessingMissingAttributes()` to surface that mistake in dev.
11. **`#[Fillable]` attribute not picked up by `Model::query()->create()` in some 13.x patch versions** — known bug [laravel/framework#59270](https://github.com/laravel/framework/issues/59270) where the reflection-based attribute registration misses the static `create()` call path on certain 13.x point releases. If you migrate from `$fillable` to `#[Fillable]` and tests start failing with `MassAssignmentException`, either pin a newer 13.x or keep `$fillable` until the patch ships. The class-level attribute is correctly honored on `new Model()` + `->save()` paths in all 13.x versions.

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

## Laravel 13 Class-Level Model Attributes

Laravel 13 ships a parallel set of class-level PHP attributes under `Illuminate\Database\Eloquent\Attributes\` that replace the legacy `$fillable` / `$hidden` / `$casts` / `$table` protected properties. The legacy properties still work (no breaking change) — the attributes are an opt-in stylistic choice for teams that prefer configuration to live above the class body.

### The full attribute cheat sheet

| Attribute | Replaces | Namespace |
|---|---|---|
| `#[Table('flights', dateFormat: 'U')]` | `$table`, `$primaryKey`, `$incrementing`, `$timestamps`, `$dateFormat` | `Illuminate\Database\Eloquent\Attributes\Table` |
| `#[Fillable(['name', 'email'])]` | `$fillable` | `Illuminate\Database\Eloquent\Attributes\Fillable` |
| `#[Hidden(['password', 'remember_token'])]` | `$hidden` | `Illuminate\Database\Eloquent\Attributes\Hidden` |
| `#[Visible(['id', 'name'])]` | `$visible` | `Illuminate\Database\Eloquent\Attributes\Visible` |
| `#[Casts(['active' => 'boolean', 'password' => 'hashed'])]` | `$casts` | `Illuminate\Database\Eloquent\Attributes\Casts` |
| `#[UsePolicy(PostPolicy::class)]` | `Gate::policy()` in `AppServiceProvider` | `Illuminate\Database\Eloquent\Attributes\UsePolicy` |

> `#[UsePolicy]` is documented in detail in `auth.md` (Policy Registration section, cycle 24).

### Full example

```php
use Illuminate\Database\Eloquent\Attributes\Casts;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Attributes\Hidden;
use Illuminate\Database\Eloquent\Attributes\Table;
use Illuminate\Database\Eloquent\Attributes\Visible;

#[Table('users')]                                              // optional — only if non-conventional table name
#[Fillable(['name', 'username', 'email', 'password'])]
#[Hidden(['password', 'remember_token'])]
#[Visible(['id', 'name', 'email'])]                            // use Visible OR Hidden, not both
#[Casts([
    'email_verified_at' => 'datetime',
    'password'          => 'hashed',
    'is_admin'          => 'boolean',
    'settings'          => 'array',
])]
class User extends Authenticatable
{
    public function posts(): HasMany
    {
        return $this->hasMany(Post::class);
    }

    // business logic only — no protected $properties clutter
}
```

### When to prefer attributes over protected properties

1. **Colocation of configuration with imports** — class body is reserved for behaviour, configuration sits above alongside the `use` block where it can be grep'd alongside the namespace. Reads like a "schema declaration".
2. **PHPStan / Psalm / Rector compatibility** — class-level attributes are first-class PHP AST nodes. Static analyzers can verify that `#[Fillable(['emial'])]` only references actual column names (with a Rector rule), whereas a typo in `$fillable = ['emial']` is invisible until runtime.
3. **Inheritance is cleaner** — child classes can re-declare the attribute. Protected properties force you to copy/paste the parent's `$fillable` array into the child.
4. **No magic property precedence** — with both `$fillable` AND `#[Fillable]` set, the attribute wins. With attributes only, the framework's reflection cache is the single source of truth.

### When to stick with protected properties

1. **You're on Laravel 12 or below** — the attributes are a Laravel 13 addition.
2. **You rely on dynamic `$fillable` mutation** (e.g. `$this->fillable = array_merge(parent::$fillable, ['legacy_field'])`). Attributes are resolved once via reflection at boot.
3. **You use a code-style tool that bans class-level attributes** (rare, but some shops have a rule).

### Pitfalls

- **Known bug with `Model::query()->create()` on some 13.x patches** ([laravel/framework#59270](https://github.com/laravel/framework/issues/59270)) — the reflection-based attribute registration misses the static `create()` query-builder path on certain point releases. `new Model()` + `->save()` is always fine. If migrating from `$fillable`, keep an eye on your test suite and pin a patched 13.x.
- **Don't mix `#[Visible]` and `$hidden`** — pick one. If both are set, the attribute wins and the property is ignored.
- **Class-level attributes are NOT inherited by default** — a child model without `#[Fillable]` has zero fillable columns. Re-declare on each child, or use the property.

See [Laravel 13 Eloquent docs — Mutators & Casting](https://laravel.com/docs/13.x/eloquent-mutators) for the full API reference.

## Vector Similarity Search (Laravel 13+)

Laravel 13 ships first-party **semantic / vector search** built on PostgreSQL + `pgvector`. No third-party wrapper, no raw SQL — `whereVectorSimilarTo()` works on both `DB::table()` and the Eloquent query builder, accepts a plain string, embeds it through the Laravel AI SDK, runs cosine similarity against a `vector` column, and returns a relevance-sorted Eloquent collection.

### Setup

```php
// 1. Ensure the pgvector extension is installed
// In a migration:
public function up(): void
{
    Schema::ensureVectorExtensionExists();   // runs CREATE EXTENSION IF NOT EXISTS vector

    Schema::create('documents', function (Blueprint $table) {
        $table->id();
        $table->string('title');
        $table->text('body');
        $table->vector('embedding', dimensions: 1536)->index();   // HNSW index, cosine distance
        $table->timestamps();
    });
}
```

### Cast + similarity query

```php
use Laravel\Ai\Embeddings;
use App\Models\Document;

// 2. Cast the embedding column on the model so it round-trips as a PHP array
class Document extends Model
{
    protected function casts(): array
    {
        return ['embedding' => 'array'];   // ['embedding' => 'array'] in #[Casts(...)] form too
    }
}

// 3. Generate embeddings (one-time per chunk)
$texts = Document::pluck('body')->all();
$embeddings = Embeddings::for($texts)->generate();   // uses OpenAI / Gemini / etc.
foreach ($embeddings->embeddings as $i => $vector) {
    Document::find($i + 1)->update(['embedding' => $vector]);
}

// 4. Run a similarity search — accepts a STRING, embeds it, ranks results
$results = Document::query()
    ->whereVectorSimilarTo('embedding', 'Best wineries in Napa Valley', minSimilarity: 0.3)
    ->limit(10)
    ->get();
// SQL (Postgres): SELECT * FROM documents ORDER BY embedding <=> $query_embedding LIMIT 10

// Works on DB::table() too
$rows = DB::table('documents')
    ->whereVectorSimilarTo('embedding', 'Best wineries in Napa Valley')
    ->limit(10)
    ->get();
```

### `minSimilarity` semantics

`minSimilarity` is a cosine similarity between `0.0` and `1.0`:
- `1.0` = identical vectors
- `0.0` = orthogonal / unrelated
- Default threshold (no value passed) is `0.0` — returns all rows ranked. Always pass a real threshold in production or you'll fetch the entire table.

### Backfill pattern (don't block writes)

```php
// app/Jobs/BackfillDocumentEmbeddings.php
class BackfillDocumentEmbeddings implements ShouldQueue
{
    use Queueable;

    public function handle(): void
    {
        Document::whereNull('embedding')
            ->chunkById(100, function ($docs) {
                $texts = $docs->pluck('body')->all();
                $resp = Embeddings::for($texts)->generate();
                foreach ($docs as $i => $doc) {
                    $doc->update(['embedding' => $resp->embeddings[$i]]);
                }
            });
    }
}
```

### Common pitfalls

- **Forgot the cast** — without `'embedding' => 'array'`, the attribute returns the raw `[0.05,0.10,...]` string and your downstream code breaks. Add the cast in step 2.
- **Forgot the pgvector extension** — query fails with `extension "vector" is not available`. Use `Schema::ensureVectorExtensionExists()` in a migration so it's reproducible.
- **Dimension mismatch** — OpenAI `text-embedding-3-small` is 1536 dims; `text-embedding-3-large` is 3072. Match your `->vector('embedding', dimensions: N)` column to the model you embed with, or pgvector rejects the insert.
- **No HNSW index** — `->index()` on the vector column creates the HNSW index. Without it, similarity search is O(N) brute-force; with it, O(log N) approximate nearest-neighbor. Below ~10k rows you can skip it; above, you need it.
- **Passing a `null` similarity threshold** — `minSimilarity: 0` returns everything. Always set a real floor (typically 0.3–0.5 depending on domain).

For a full production walkthrough (RAG agent, hybrid BM25 + vector, streaming), see `ai.md` (Laravel AI SDK vs MCP vs Boost section).

## Automatic Eager Loading (Laravel 12.8+)

The fourth `Model::` boot switch (alongside `preventLazyLoading` / `preventSilentlyDiscardingAttributes` / `preventAccessingMissingAttributes`):

```php
// AppServiceProvider::boot()
Model::automaticallyEagerLoadRelationships();
```

### What it does

When this is enabled, Laravel watches for relation access *on a model inside a collection of models*. The moment any relation is touched on any model in the collection, Laravel automatically issues a single `WHERE IN` query to fetch that relation for the entire collection — *not* per-row.

```php
// With automaticallyEagerLoadRelationships() enabled:

$users = User::all();   // 1 query, no with()

// Later, somewhere deep in the view:
@foreach ($users as $user)
    {{ $user->posts->count() }}   // ← first access triggers a single SELECT ... WHERE user_id IN (...)
@endforeach
// Result: 2 queries total, not 1001.
```

### When to use it vs `Model::preventLazyLoading()`

| Mode | Behaviour | Use when |
|---|---|---|
| `Model::preventLazyLoading(true)` | Throws on lazy access | Dev / CI — fail-fast on forgotten `with()` |
| `Model::automaticallyEagerLoadRelationships()` | Silently fixes the N+1 | Production — keep N+1s from sneaking into dashboards |
| **Both** | Throws in dev, auto-fixes in prod | **Recommended default** — see AppServiceProvider block below |

### Recommended combined setup

```php
// AppServiceProvider::boot()
public function boot(): void
{
    Model::preventLazyLoading(! app()->isProduction());         // fail-fast in dev
    Model::automaticallyEagerLoadRelationships();               // safety net in prod

    Model::preventSilentlyDiscardingAttributes(! app()->isProduction());
    Model::preventAccessingMissingAttributes(! app()->isProduction());
    Model::prohibitDestructiveCommands(
        ! app()->isProduction() && ! app()->runningInConsole()
    );
}
```

### Limitations

- **Only fires for relations accessed on a model inside a collection** — `$user->posts` on a single model is still lazy. (But `$user->posts` on a single model is just one query, so no N+1.)
- **Only loads what was actually accessed** — if you access `$user->posts` then later `$user->comments`, you get two queries, not one combined eager load. Use `with(['posts', 'comments'])` explicitly for known relations.
- **Doesn't help with `select()`-clipped columns** — if you `User::select(['id', 'name'])->get()`, accessing `$user->email` still triggers a missing-attribute error (and the auto-eager-loader won't fix it). Pair with `Model::preventAccessingMissingAttributes()` to catch that case in dev.
- **First-touch latency cost** — the very first access adds one query (the auto-fetch), so dashboards that touch 10 different relations pay 10 queries serially. If you know the relations up front, explicit `->with([...])` is still faster.

## Updated from Research (2026-07-06, cycle 28)

- **Laravel 13 Class-Level Model Attributes (`#[Table]` / `#[Fillable]` / `#[Hidden]` / `#[Visible]` / `#[Casts]`)** — added a new top-level "Laravel 13 Class-Level Model Attributes" section documenting the full attribute cheat sheet under `Illuminate\Database\Eloquent\Attributes\`. Cross-linked to `auth.md` for `#[UsePolicy]` (already documented in cycle 24). The attributes are an opt-in stylistic alternative to `$fillable` / `$hidden` / `$casts` / `$table` / `$primaryKey` — they live above the class body and are first-class PHP AST nodes (PHPStan/Psalm/Rector-compatible). Includes the known bug [laravel/framework#59270](https://github.com/laravel/framework/issues/59270) where `Model::query()->create()` misses the attribute registration on certain 13.x point releases.
- **Vector Similarity Search (`whereVectorSimilarTo`, Laravel 13+)** — added a new top-level "Vector Similarity Search" section covering `pgvector` setup (`Schema::ensureVectorExtensionExists()`), `->vector('embedding', dimensions: N)->index()` migrations, the required `'embedding' => 'array'` cast, `whereVectorSimilarTo('embedding', $queryString, minSimilarity: 0.3)` query usage on both `Eloquent` and `DB::table()`, `minSimilarity` semantics (0.0–1.0 cosine), and a queued `chunkById` backfill pattern. Cross-linked to `ai.md` for the RAG agent / hybrid BM25+vector walkthrough. This was the largest gap in `eloquent.md` — first-party semantic search shipped in Laravel 13 and was completely undocumented.
- **`Model::automaticallyEagerLoadRelationships()` (Laravel 12.8+)** — added a new top-level "Automatic Eager Loading" section covering the auto-N+1-prevention boot switch, the when-to-use-it vs `preventLazyLoading` matrix, a recommended combined `AppServiceProvider::boot()` block (preventLazy in dev + auto-eager in prod), and four explicit limitations (only fires on collection members, only loads accessed relations, doesn't help with `select()`-clipped columns, first-touch latency cost).
- **`whereAll()` / `whereAny()` / `orWhereAll()` / `orWhereAny()` (Laravel 10.47+)** — added a "Multi-column whereAll / whereAny" block in the Query Builder section. Multi-column WHERE with AND/OR semantics — cleaner alternative to nested closures for flat column lists. Includes when-to-pick-it vs `orWhere(fn)` rationale.
- **`whereRelation` and `whereMorphRelation` (Laravel 10+/12+)** — the `whereMorphRelation` Laravel 12 addition was missing from the existing `whereRelation` mention. Added both with examples.
- **Common Mistakes list grew 7 → 11 entries** — added "Forgetting `embedding` cast on `vector` columns", "Using `whereVectorSimilarTo` without `pgvector` enabled", "Eager-loading the world with `automaticallyEagerLoadRelationships()`", and the `#[Fillable]` `Model::query()->create()` bug.
- **No `versions.md` release-notes change** — `v13.18.1` (2026-07-02) remains head of `13.x`. No new framework release since cycle 27. No new PHP batch. No new Laravel CVE.

---


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
