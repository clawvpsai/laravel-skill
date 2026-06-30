# Database Migrations — Schema Management, Seeding, Schema Builder

## Migration Structure

```bash
php artisan make:migration create_posts_table
# Creates: database/migrations/xxxx_create_posts_table.php
```

```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('posts', function (Blueprint $table) {
            $table->id();                          // auto-increment PK
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->string('title');
            $table->text('body')->nullable();
            $table->timestamp('published_at')->nullable();
            $table->timestamps();                  // created_at + updated_at
            $table->softDeletes();                 // deleted_at

            $table->index(['user_id', 'published_at']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('posts');
    }
};
```

## Column Types

| Type | Notes |
|---|---|
| `$table->id()` | BigInt auto-increment, primary key |
| `$table->foreignId('x_id')` | BigInt FK, adds index + FK constraint |
| `$table->foreignUuidFor(Model::class)` | UUID FK (CHAR(36)), adds index + FK constraint |
| `$table->string('x', 255)` | VARCHAR with optional length |
| `$table->text('x')` | TEXT |
| `$table->longText('x')` | LONGTEXT |
| `$table->integer('x')` | INT |
| `$table->bigInteger('x')` | BIGINT |
| `$table->float('x', 8, 2)` | FLOAT(8,2) |
| `$table->decimal('x', 10, 2)` | DECIMAL(10,2) — use for money, not FLOAT |
| `$table->boolean('x')` | TINYINT(1) |
| `$table->date('x')` | DATE |
| `$table->dateTime('x')` | DATETIME |
| `$table->timestamp('x')` | TIMESTAMP |
| `$table->json('x')` | JSON |
| `$table->uuid('x')` | CHAR(36) |
| `$table->enum('x', ['a','b'])` | ENUM (use with caution) |
| `$table->morphs('taggable')` | taggable_id + taggable_type indexes |
| `$table->nullableMorphs('taggable')` | nullable version |

## UUID Foreign Keys (Laravel 13.9+)

Use `foreignUuidFor()` for UUID-based foreign key columns — cleaner than manually adding `$table->uuid()` + `$table->foreign()`:

```php
Schema::create('comments', function (Blueprint $table) {
    $table->id();
    $table->foreignUuidFor(User::class);         // user_uuid — adds FK + index
    $table->foreignUuidFor(Post::class, 'post'); // post_uuid — custom suffix
    $table->text('body');
    $table->timestamps();
});
```

```php
// Equivalent manual approach (before Laravel 13.9):
$table->uuid('user_uuid');
$table->foreign('user_uuid')->references('id')->on('users');
// foreignUuidFor() collapses this into one call
```

**With constraints:**
```php
$table->foreignUuidFor(User::class)->nullable()->constrained()->cascadeOnDelete();
```

## Modifiers

```php
$table->string('email')->unique();           // unique index
$table->string('title')->default('Untitled');
$table->string('token')->nullable();
$table->integer('votes')->unsigned();
$table->string('name')->charset('utf8mb4');
$table->string('slug')->collation('utf8mb4_unicode_ci');
$table->text('bio')->useCurrent();            // DEFAULT CURRENT_TIMESTAMP (on insert)
$table->timestamp('updated_at')->useCurrentOnUpdate(); // auto-update on update
$table->json('meta')->default([]);            // default value as expression
$table->string('notes')->comment('audit log');  // COMMENT ON COLUMN (MySQL/MariaDB/PostgreSQL)
```

## Timestamps & Auto-Update

```php
// Timestamps with automatic management
$table->timestamps();       // created_at + updated_at (nullable, auto-set by Laravel)
$table->timestamp('verified_at')->nullable();
$table->softDeletes();      // deleted_at column

// Explicit default values (Laravel 13+)
$table->timestamp('created_at')->useCurrent();              // DEFAULT CURRENT_TIMESTAMP
$table->timestamp('updated_at')->useCurrentOnUpdate();      // ON UPDATE CURRENT_TIMESTAMP

// Explicit nullable timestamps (Laravel 11+ default — no longer auto-not-null)
$table->timestamp('published_at')->nullable();  // must be nullable if not always set
```

### ⚠️ Y2K38 / 2038 Problem — Use `dateTime()` for Future-Dated Records

MySQL's `TIMESTAMP` type stores a 32-bit Unix epoch and **overflows on 2038-01-19 03:14:07 UTC**.
Laravel's default `timestamps()` emits `TIMESTAMP` columns, which is fine for `created_at` /
`updated_at` (always "now") but **breaks** for any column that may legitimately hold a future date
(subscription expirations, scheduled publishing, calendar events, audit retention beyond 2038, etc.).
Active community discussion: [laravel/framework#60593](https://github.com/laravel/framework/discussions/60593)
(posted 2026-06-26, "Unanswered" as of this writing).

```php
// ❌ WRONG — overflows past 2038-01-19 03:14:07 UTC
$table->timestamp('expires_at')->nullable();     // MySQL TIMESTAMP, 32-bit
$table->timestamp('scheduled_for')->nullable();
$table->timestamp('published_after')->nullable();

// ✅ RIGHT — DATETIME stores year up to 9999, no 2038 boundary
$table->dateTime('expires_at')->nullable();
$table->dateTime('scheduled_for')->nullable();
$table->dateTime('published_after')->nullable();

// ✅ RIGHT — `timestamps()` itself is safe (always stores NOW); only user-defined future
//    date columns need DATETIME. Document this in code review checklists.
```

**Practical rule:** if a column can ever hold a date more than ~12 years in the future
(subscriptions, scheduling, regulatory retention, calendars, contracts, wills, leases),
use `dateTime()` not `timestamp()`. Audit existing schemas with:
```bash
grep -rE 'timestamp\([a-z_]+\)' database/migrations/ | grep -v 'created_at\|updated_at\|deleted_at\|useCurrent'
```
Note: on PostgreSQL `TIMESTAMP` is already 64-bit and unaffected by the 2038 boundary — the
problem is MySQL/MariaDB specific. SQLite uses strings and is also unaffected.

## Generated Columns & Identity Columns

**MySQL/MariaDB — `storedAs()` / `virtualAs()`:**
```php
// STORED generated column — physically persisted, can be indexed
$table->string('full_name')->storedAs("CONCAT(first_name, ' ', last_name)");

// VIRTUAL generated column — computed on the fly, not persisted (saves space)
$table->decimal('discounted_price')->virtualAs('price * 0.9');
```

**PostgreSQL — `storedAs()` only** (PostgreSQL has no virtual columns; use views or generated columns for computed values). PostgreSQL also has `generatedAs()` for **identity columns** (SQL-standard auto-increment, preferred over the legacy `SERIAL`):

```php
// BIGINT GENERATED ALWAYS AS IDENTITY — SQL standard
$table->bigIncrements('id');          // Laravel 13.13+ defaults to this on PostgreSQL
$table->bigInteger('seq')->generatedAs();                          // always
$table->bigInteger('seq')->generatedAs('ALWAYS AS IDENTITY');      // explicit
$table->bigInteger('seq')->generatedAs('BY DEFAULT', 'START WITH 1000 INCREMENT BY 5');
// Chain ->always() to enforce sequence over input, ->primary() etc.
$table->bigInteger('seq')->generatedAs()->always()->primary();
```

**SQLite — `storedAs()` only.** No `virtualAs()` support, no `generatedAs()` for identity.

**Common gotcha — `generatedAs` is for *identity* columns only** (PostgreSQL `smallint`/`integer`/`bigint` sequences). It will fail with "syntax error" if you try to use it for a string concatenation. Use `storedAs()` for derived string/numeric values.

## Indexes

```php
$table->index(['user_id', 'created_at']);      // composite B-tree index
$table->unique(['email']);                     // unique constraint
$table->primary('id');                         // explicit PK
$table->foreignId('user_id')->constrained();   // FK with index
$table->dropIndex(['user_id']);                // remove index
$table->dropUnique(['email']);                 // remove unique constraint

// Naming convention — keep under 64 chars for MySQL
$table->index(['user_id', 'created_at'], 'ix_posts_user_created');
```

### Available Index Types (Laravel 13.x)

| Type | Method | Supported on |
|---|---|---|
| Primary key | `$table->primary('id')` | All |
| Unique | `$table->unique('email')` | All |
| Plain (B-tree) | `$table->index('col')` | All |
| Fulltext | `$table->fullText('body')` | MariaDB / MySQL / PostgreSQL |
| Fulltext (named language) | `$table->fullText('body')->language('english')` | PostgreSQL only |
| Spatial | `$table->spatialIndex('location')` | MariaDB / MySQL (PostGIS uses GIST, not Blueprint) |
| Composite | `$table->index(['a', 'b'])` | All |

```php
// Fulltext search (e.g., blog posts)
$table->fullText(['title', 'body']);  // MySQL/MariaDB need InnoDB 5.6+; PG needs tsvector column

// Spatial index (MySQL — requires NOT NULL geometry column with SRID restriction in 8.0+)
$table->geometry('location', subtype: 'point', srid: 4326);
$table->spatialIndex('location');

// Drop fulltext
$table->dropFullText(['title', 'body']);
```

**Practical rules:**
- Fulltext is useless for short strings (emails, slugs, tags) — use B-tree `unique` instead
- Don't fulltext-index columns you'll only filter on with `=` (use plain index)
- PostgreSQL fulltext needs `tsvector` column or `to_tsvector()` expression index; Laravel's `fullText()` works on the table column but for weighted search use `whereRaw("to_tsvector('english', body) @@ plainto_tsquery('english', ?)", [$q])`
- Spatial indexes require SRID-aware columns on MySQL 8.0+; older MySQL 5.7 syntax silently differs

## Renaming & Changing Columns

```php
Schema::table('posts', function (Blueprint $table) {
    $table->renameColumn('bio', 'biography');
});

// Drop column (Laravel 13+: requires no other column to depend on it via FK)
Schema::table('posts', function (Blueprint $table) {
    $table->dropColumn('bio');
});
```

**Requires `doctrine/dbal`:**
```bash
composer require doctrine/dbal
```

## Foreign Key Constraints

```php
$table->foreignId('user_id')
    ->constrained('users')
    ->cascadeOnDelete()      // drop row when parent deleted
    ->cascadeOnUpdate()      // update FK when parent PK updated
    ->restrictOnDelete()    // prevent delete of parent with children
    ->nullOnDelete();       // set FK to null on parent delete

// On existing column
$table->foreign('user_id')
    ->references('id')
    ->on('users')
    ->cascadeOnDelete();

// Drop FK
$table->dropForeign(['user_id']);
```

## Running Migrations

```bash
php artisan migrate                  # run pending migrations
php artisan migrate --force         # run in production (skip confirmation)
php artisan migrate:rollback         # rollback last batch
php artisan migrate:rollback --step=3
php artisan migrate:fresh            # drop all tables + re-run (dev only!)
php artisan migrate:refresh          # rollback + migrate
php artisan migrate:status           # show which migrations ran
php artisan migrate --path=database/migrations/custom
```


## Schema Dump — `--without-migration-data` (Laravel 13.17+)

`php artisan schema:dump` normally emits one SQL file per migration plus a `_migration_data.sql` file containing the `INSERT` statements for any rows inserted during migrations (lookup tables, seed data baked in). In 13.17 (PR #60570), the new `--without-migration-data` flag emits **schema only** — no `INSERT` statements — even when migration data is present:

```bash
# Default behavior — schema + any migration-time INSERT statements
php artisan schema:dump

# Schema only — drops the migration-data file
php artisan schema:dump --without-migration-data

# Combine with --prune to remove migrations after dumping
php artisan schema:dump --without-migration-data --prune
```

**When to use it:**
- Greenfield DB provisioning (no seed data should ship with schema)
- Shipping schema to read replicas that should only contain replicated data
- Compliance environments where migration-time inserts must not be auditable separately from application seeds
- Schema files distributed as composer packages (the `vendor:publish` target should be pure DDL)

**Audit checklist:** if your existing dumps contain a `_migration_data.sql` and you switch to `--without-migration-data`, verify the missing rows were not relied upon by the running app (they would still be inserted by `migrate` itself when you re-run migrations during deploy).

See `deployment.md` (Postgres Transaction Pooler Support / Schema section) for the deploy-side flow.


## Migration Lifecycle Events (Laravel 13.9+)

`MigrationStarted` and `MigrationEnded` events now carry the migration class name (not just connection name) for granular event handling:

```php
use Illuminate\Database\Events\MigrationStarted;
use Illuminate\Database\Events\MigrationEnded;
use Illuminate\Support\Facades\Event;

Event::listen(MigrationStarted::class, function (MigrationStarted $event) {
    Log::info("Migration starting", [
        'connection' => $event->connectionName,  // e.g., 'mysql'
        'method' => $event->method,             // 'up' or 'down'
    ]);
});

Event::listen(MigrationEnded::class, function (MigrationEnded $event) {
    Log::info("Migration completed", [
        'connection' => $event->connectionName,
        'method' => $event->method,
    ]);
});
```

**Filter by specific migration (by class name):**
```php
Event::listen(MigrationEnded::class, function (MigrationEnded $event) {
    // Laravel 13.9+ events include migration name in event
    // Check the event's migration class name if available
    if ($event->method === 'down') {
        // Schema changed — invalidate caches
        Cache::flush();
    }
});
```

**Use cases:**
- Audit logging — track which migrations ran and when
- Alerting — notify on migration start/completion in deployment pipelines
- Resource management — warm up connections before migration, release after
- Post-migration cache clearing — `config:cache` after schema changes

**Post-migration cache clearing:**
```php
Event::listen(MigrationEnded::class, function (MigrationEnded $event) {
    if ($event->method === 'up') {
        Artisan::call('config:cache');
        Artisan::call('route:cache');
    }
});
```

## Seeding

```bash
php artisan make:factory PostFactory --model=Post
php artisan make:seeder PostSeeder
```

**Factory:**
```php
namespace Database\Factories;

use App\Models\Post;
use Illuminate\Database\Eloquent\Factories\Factory;

class PostFactory extends Factory
{
    protected $model = Post::class;

    public function definition(): array
    {
        return [
            'title'   => fake()->sentence(),
            'body'    => fake()->paragraphs(3, true),
            'user_id' => User::factory(),
            'published_at' => fake()->optional()->dateTime(),
        ];
    }

    public function unpublished(): static
    {
        return $this->state(fn() => ['published_at' => null]);
    }
}
```

**Seeder:**
```php
namespace Database\Seeders;

use App\Models\Post;
use App\Models\User;
use Illuminate\Database\Seeder;

class PostSeeder extends Seeder
{
    public function run(): void
    {
        User::factory(10)->create();

        Post::factory(50)->create()->each(fn($post) =>
            $post->update(['published_at' => now()])
        );
    }
}
```

**Run seeders:**
```bash
php artisan db:seed                    # run all seeders
php artisan db:seed --class=PostSeeder
php artisan migrate:fresh --seed       # reset + seed
```

## Common Mistakes

1. **`migrate:fresh` in production** — drops ALL tables, no undo
2. **No `down()` method** — migrations must be reversible
3. **Changing column without doctrine/dbal** — requires the package
4. **Long index names** — MySQL max 64 chars; use `$table->index('col', 'short_name')`
5. **Enum as FK** — don't use enum for foreign keys, use integer IDs
6. **No timestamps on pivot tables** — many2many needs `timestamps()` for `withTimestamps()`
7. **`decimal` vs `float` for money** — always use `decimal(10,2)` for currency, never float
8. **Nullable timestamps without default** — Laravel expects `nullable()` timestamps to have a default or be set explicitly; can cause "Incorrect datetime value" errors
9. **Using `timestamp()` for future-dated columns** — MySQL `TIMESTAMP` overflows at 2038-01-19 03:14:07 UTC. Use `dateTime()` for `expires_at`, `scheduled_for`, retention dates, calendar events, or anything that may legitimately hold a date > 12 years out. See Y2K38 section above. ([laravel/framework#60593](https://github.com/laravel/framework/discussions/60593))

## Updated from Research (2026-06-26, cycle 5)

- **`schema:dump --without-migration-data` (Laravel 13.17+)** — PR #60570 by @jackbayliss emits **schema only** (no `INSERT` statements) even when migration data is present. Combines with `--prune` to remove migrations after dumping the clean schema file. Use cases: greenfield DB provisioning, shipping schema to read replicas, compliance environments that must not ship seed data with DDL, composer-distributed schema packages. See the detailed section above.

---

## Updated from Research (2026-05-18)

- **foreignUuidFor() (Laravel 13.9+)** — `$table->foreignUuidFor(Model::class)` collapses UUID FK + index + constraint into one call; cleaner than manual `$table->uuid()` + `->foreign()`
- Laravel 13.9 adds `MigrationStarted` and `MigrationEnded` events with migration class name (previously only connection name was available)
- Laravel 13 supports `useCurrentOnUpdate()` for auto-updating timestamps
- `decimal` is the correct type for monetary values, not `float`
- `foreignId` with `constrained()` auto-detects table name from column name
- Laravel 13 drop column requires no FK dependencies on the target column

Source: [Laravel Migrations](https://laravel.com/docs/13.x/migrations) | [Laravel 13 Release Notes](https://laravel.com/docs/13.x/releases)

---

## Updated from Research (2026-06-30, cycle 15)

**migrations.md** (3.5 days stale — oldest file) — refreshed to cover the documented-but-skipped
Laravel 13 schema primitives that AI models keep asking about wrong:

- **`storedAs()` / `virtualAs()` generated columns** — MySQL/MariaDB support both `STORED` (persisted,
  indexable) and `VIRTUAL` (computed on read, no storage overhead). PostgreSQL/SQLite support
  `storedAs()` only. Critical for computed columns that should be queryable (`full_name`,
  `discounted_price`, denormalized rollups).
- **`generatedAs()` identity columns** (PostgreSQL) — SQL-standard `BIGINT GENERATED ALWAYS AS
  IDENTITY` syntax. Laravel 13.13+ defaults `bigIncrements()` to this on PostgreSQL (no more
  `SERIAL`). Common AI hallucination: trying to use `generatedAs()` for string concatenation —
  it only works for `smallint`/`integer`/`bigint` sequences. Use `storedAs()` for derived strings.
- **`->comment('text')` modifier** — adds `COMMENT ON COLUMN` (MySQL/MariaDB/PostgreSQL). Useful
  for documenting column purpose, units, or audit source in the schema itself.
- **Full index type matrix** — `fullText()` (MariaDB/MySQL/PostgreSQL with optional `->language()` on
  PG), `spatialIndex()` (MySQL/MariaDB), and the gotchas (fulltext on short strings is a waste;
  PG fulltext needs `tsvector` for weighted search).

**No new Laravel 13.x release** as of 2026-06-30 18:00 UTC. v13.17.0 (June 23, 2026) remains the
latest tagged release. v13.17.1 / v13.18.0 still pending — track
[github.com/laravel/framework/releases](https://github.com/laravel/framework/releases).
