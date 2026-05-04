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
| `$table->string('x', 255)` | VARCHAR with optional length |
| `$table->text('x')` | TEXT |
| `$table->longText('x')` | LONGTEXT |
| `$table->integer('x')` | INT |
| `$table->bigInteger('x')` | BIGINT |
| `$table->float('x', 8, 2)` | FLOAT(8,2) |
| `$table->decimal('x', 10, 2)` | DECIMAL(10,2) |
| `$table->boolean('x')` | TINYINT(1) |
| `$table->date('x')` | DATE |
| `$table->dateTime('x')` | DATETIME |
| `$table->timestamp('x')` | TIMESTAMP |
| `$table->json('x')` | JSON |
| `$table->uuid('x')` | CHAR(36) |
| `$table->enum('x', ['a','b'])` | ENUM (use with caution) |
| `$table->morphs('taggable')` | taggable_id + taggable_type indexes |
| `$table->nullableMorphs('taggable')` | nullable version |

## Modifiers

```php
$table->string('email')->unique();           // unique index
$table->string('title')->default('Untitled');
$table->string('token')->nullable();
$table->integer('votes')->unsigned();
$table->string('name')->charset('utf8mb4');
$table->string('slug')->collation('utf8mb4_unicode_ci');
$table->text('bio')->useCurrent();            // DEFAULT CURRENT_TIMESTAMP
```

## Indexes

```php
$table->index(['user_id', 'created_at']);      // composite index
$table->unique(['email']);                     // unique constraint
$table->primary('id');                         // explicit PK
$table->foreignId('user_id')->constrained();   // FK with index
$table->dropIndex(['user_id']);                // remove index
```

## Renaming & Changing Columns

```php
Schema::table('posts', function (Blueprint $table) {
    $table->renameColumn('bio', 'biography');
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
    ->restrictOnDelete()      // prevent delete of parent
    ->nullOnDelete();         // set FK to null on parent delete
```

## Running Migrations

```bash
php artisan migrate                  # run pending migrations
php artisan migrate --force          # run in production
php artisan migrate:rollback         # rollback last batch
php artisan migrate:rollback --step=3
php artisan migrate:fresh            # drop all + re-run (dev only!)
php artisan migrate:refresh         # rollback + migrate
php artisan migrate:status          # show which migrations ran
php artisan migrate --path=database/migrations/custom
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
4. **Long index names** — MySQL max 64 chars; use `$table->index('user_id', 'ix_short_name')`
5. **Enum as FK** — don't use enum for foreign keys, use integer IDs
6. **No timestamps on pivot tables** — many2many needs `timestamps()` for `laravel_through_many`

## Updated from Research (2026-05)
- Laravel 13 supports `usingCurrent()` for TIMESTAMP defaults
- `foreignId` with `constrained()` auto-detects table name from column name

Source: [Laravel Migrations](https://laravel.com/docs/13.x/migrations)
