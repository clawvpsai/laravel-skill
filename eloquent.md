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