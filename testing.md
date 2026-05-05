# Testing — PHPUnit, Factories, Feature Tests

## Setup

```bash
# Run tests
php artisan test

# With coverage (requires phpunit-xml)
php artisan test --coverage

# Run specific test
php artisan test --filter=PostTest

# Watch (with Pest)
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

## Mocking

```php
// Mock a service
$this->mock(PostService::class, function ($mock) {
    $mock->shouldReceive('process')->once()->with(5)->andReturn(true);
});

// Partial mock
$this->spy(PostService::class, function ($spy) {
    $spy->shouldReceive('process')->with(5)->andReturn(true);
});

// Mock facades
Cache::shouldReceive('get')
    ->with('posts')
    ->andReturn(collect(['post1', 'post2']));

// Mock queue
Queue::fake(); // jobs aren't actually dispatched

// Assert a job was dispatched
Queue::assertPushed(ProcessPostJob::class, fn($job) => $job->postId === 5);

// Assert job was NOT dispatched
Queue::assertNotPushed(ProcessPostJob::class);
```

## HTTP Client Testing (Guzzle Mocking)

```php
// Fake HTTP responses
Http::fake([
    'api.github.com/*' => Http::response(['name' => 'laravel'], 200),
    'stripe.com/*' => Http::response(['id' => 'ch_123'], 500), // test failure
]);

// In your test
$response = $this->post('/webhook/stripe', ['event' => 'payment']);
$response->assertStatus(200);

// Assert HTTP was called
Http::assertSent(function ($request) {
    return $request->url() === 'https://api.github.com/user';
});
```

## Laravel 13 Testing Attributes

Laravel 13 expands PHP attributes for testing:

```php
use Illuminate\Testing\Attributes\Group;
use Illuminate\Testing\Attributes\TestProperty;
use Illuminate\Testing\Attributes\UnitTest;

#[Group('feature')]
#[TestProperty('scenario', 'user-authentication')]
class AuthenticationTest extends TestCase
{
    // Group tests and add metadata for test reporting/filtering

    #[UnitTest]
    public function test_password_hashing(): void
    {
        // Skips booting Laravel app — faster for pure unit tests
        $hash = Hash::make('secret');
        $this->assertTrue(Hash::check('secret', $hash));
    }
}
```

**Key attributes:**
- `#[Group('name')]` — organize tests into groups for filtering (`php artisan test --group=feature`)
- `#[TestProperty('key', 'value')]` — attach metadata to tests for reporting
- `#[UnitTest]` — skip Laravel app boot for faster pure unit tests that don't need it

## Bulk JSON Path Assertions (Laravel 13.7+)

TestResponse supports asserting multiple JSON paths at once:

```php
$response = $this->getJson('/api/posts/1');

$response->assertJsonPaths([
    'data.id',
    'data.title',
    'data.author.id',
    'data.author.name',
]);
```

This is cleaner than chaining multiple `assertJsonPath` calls when you need to verify many fields.

## Pest PHP (Popular Laravel Testing Framework)

```bash
composer require pestphp/pest --dev
php artisan pest:install
```

```php
// tests/Feature/PostTest.php (uses Pest instead of PHPUnit)
use Illuminate\Foundation\Testing\RefreshDatabase;

pest()->describe('Post Controller', function () {
    beforeEach(fn() => RefreshDatabase::refreshDatabase());
    
    it('can list published posts', function () {
        $post = Post::factory()->create(['published_at' => now()]);
        
        $response = getJson('/api/posts');
        
        $response->assertSuccessful()
            ->assertJsonCount(1, 'data');
    });
    
    it('rejects unauthenticated create', function () {
        $response = postJson('/api/posts', ['title' => 'Test']);
        
        $response->assertStatus(401);
    });
});
```

**Run Pest:**
```bash
php artisan pest           # all tests
php artisan pest --group=feature  # specific group
php artisan pest --filter="can list"  # filter by name
```

**Pest Plugins:**
```bash
composer require pestphp/pest-plugin-laravel --dev
php artisan pest --coverage  # coverage report
```

## Laravel Dusk (Browser Testing)

```bash
composer require --dev laravel/dusk
php artisan dusk:install
```

```php
// tests/Browser/PostCreationTest.php
use Laravel\Dusk\Browser;
use Tests\DuskTestCase;

class PostCreationTest extends DuskTestCase
{
    public function test_user_can_create_post(): void
    {
        $this->browse(function (Browser $browser) {
            $browser->loginAs(User::factory()->create())
                ->visit('/posts/create')
                ->type('title', 'My Post')
                ->type('body', 'Post content here')
                ->press('Publish')
                ->assertPathIs('/posts');
        });
    }
}
```

## Common Mistakes

1. **Not using `RefreshDatabase`** — tests see stale data from previous tests
2. **Interacting with real services** — mock mail, queue, external APIs
3. **Hardcoding IDs** — use factories, IDs change between DBs
4. **Testing too much in one test** — one assertion per test is fine, be specific
5. **Not testing edge cases** — empty, null, very long strings, special characters
6. **Skipping unit tests** — too many things break silently without them
7. **`assertStatus(200)` vs `assertSuccessful()`** — Laravel 11+ prefers `assertSuccessful()` for clearer intent
8. **Not faking queue jobs** — `Queue::fake()` prevents actual job dispatch during tests
9. **Forgetting to mock HTTP** — real API calls slow tests and can fail unexpectedly

## Updated from Research (2026-05)

- **Laravel 13 Testing Attributes** — `#[Group]` and `#[TestProperty]` for organizing/filtering tests, `#[UnitTest]` to skip Laravel app boot for pure unit tests.
- **Bulk JSON Path Assertions (13.7+)** — `assertJsonPaths()` for asserting multiple JSON paths in one call.
- **Pest PHP** — increasingly standard in Laravel ecosystem; `pest()` function replaces `TestCase` class methods.
- **HTTP Client Mocking** — `Http::fake()` for mocking external API calls in tests.
- **Mocking Best Practices** — use `mock()` for services, `spy()` when you only need to verify calls happened, `Queue::fake()` for job queues.

Source: [Laravel 13 Docs - Testing](https://laravel.com/docs/13.x/testing)