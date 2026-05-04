# Testing — PHPUnit, Factories, Feature Tests

## Setup

```bash
# Run tests
php artisan test

# With coverage (requires phpunit-xml)
php artisan test --coverage

# Run specific test
php artisan test --filter=PostTest

# Watch (with stakes/pest)
./vendor/bin/pest --watch
```

## PHPUnit Config

```xml
<!-- phpunit.xml -->
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="vendor/phpunit/phpunit/phpunit.xsd"
         bootstrap="vendor/autoload.php"
         colors="true">
    <testsuites>
        <testsuite name="Unit">
            <directory suffix="Test.php">./tests/Unit</directory>
        </testsuite>
        <testsuite name="Feature">
            <directory suffix="Test.php">./tests/Feature</directory>
        </testsuite>
    </testsuites>
    <source>
        <include>
            <directory suffix=".php">./app</directory>
        </include>
    </source>
    <php>
        <env name="APP_ENV" value="testing"/>
        <env name="BCRYPT_ROUNDS" value="4"/>
        <env name="CACHE_DRIVER" value="array"/>
        <env name="DB_CONNECTION" value="sqlite"/>
        <env name="DB_DATABASE" value=":memory:"/>
        <env name="MAIL_MAILER" value="array"/>
        <env name="QUEUE_CONNECTION" value="sync"/>
        <env name="SESSION_DRIVER" value="array"/>
    </php>
</phpunit>
```

**SQLite in-memory for tests — no real DB needed.**

## Factories

```php
// database/factories/PostFactory.php
namespace Database\Factories;

use App\Models\Post;
use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;

class PostFactory extends Factory
{
    protected $model = Post::class;

    public function definition(): array
    {
        return [
            'title' => fake()->sentence(),
            'body' => fake()->paragraphs(3, true),
            'author_id' => User::factory(),
            'published_at' => fake()->optional()->dateTimeBetween('-1 year', 'now'),
        ];
    }

    public function unpublished(): static
    {
        return $this->state(fn(array $attrs) => ['published_at' => null]);
    }

    public function ownedBy(User $user): static
    {
        return $this->state(fn(array $attrs) => ['author_id' => $user->id]);
    }
}
```

**Use in tests:**
```php
$post = Post::factory()->create();
$post = Post::factory()->unpublished()->make();
$posts = Post::factory()->count(5)->create();

// With relations
$user = User::factory()->create();
$post = Post::factory()->ownedBy($user)->create();
```

## Feature Tests

```php
// tests/Feature/PostControllerTest.php
namespace Tests\Feature;

use App\Models\Post;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class PostControllerTest extends TestCase
{
    use RefreshDatabase; // reset DB between tests

    public function test_index_returns_published_posts(): void
    {
        // Arrange
        $post = Post::factory()->create(['published_at' => now()]);
        Post::factory()->unpublished()->create();

        // Act
        $response = $this->getJson('/api/posts');

        // Assert
        $response->assertSuccessful()
            ->assertJsonCount(1, 'data');
    }

    public function test_unauthenticated_user_cannot_create_post(): void
    {
        $response = $this->postJson('/api/posts', [
            'title' => 'Hello',
            'body' => 'Content',
        ]);

        $response->assertStatus(401);
    }

    public function test_post_creation_validates_input(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)->postJson('/api/posts', [
            'title' => '', // invalid
            'body' => 'short', // too short
        ]);

        $response->assertStatus(422)
            ->assertJsonValidationErrors(['title', 'body']);
    }

    public function test_user_can_update_own_post(): void
    {
        $user = User::factory()->create();
        $post = Post::factory()->ownedBy($user)->create();

        $response = $this->actingAs($user)->putJson("/api/posts/{$post->id}", [
            'title' => 'Updated Title',
        ]);

        $response->assertStatus(200);
        $this->assertDatabaseHas('posts', ['id' => $post->id, 'title' => 'Updated Title']);
    }
}
```

## Unit Tests

```php
// tests/Unit/PostTest.php
namespace Tests\Unit;

use App\Models\Post;
use PHPUnit\Framework\TestCase;

class PostTest extends TestCase
{
    public function test_excerpt_is_limited_to_120_chars(): void
    {
        $post = new Post(['body' => str_repeat('a', 200)]);
        $this->assertEquals(120, strlen($post->excerpt));
    }

    public function test_post_is_draft_without_published_at(): void
    {
        $post = new Post(['published_at' => null]);
        $this->assertFalse($post->isPublished());
    }
}
```

## Asserting Against Database

```php
// Assert record exists
$this->assertDatabaseHas('posts', ['id' => $post->id]);

// Assert record missing
$this->assertDatabaseMissing('posts', ['id' => $deletedId]);

// Assert count
$this->assertDatabaseCount('posts', 5);
```

## Common Mistakes

1. **Not using `RefreshDatabase`** — tests see stale data from previous tests
2. **Interacting with real services** — mock mail, queue, external APIs
3. **Hardcoding IDs** — use factories, IDs change between DBs
4. **Testing too much in one test** — one assertion per test is fine, be specific
5. **Not testing edge cases** — empty, null, very long strings, special characters
6. **Skipping unit tests** — too many things break silently without them
7. **`assertStatus(200)` vs `assertSuccessful()`** — Laravel 11+ prefers `assertSuccessful()` for clearer intent