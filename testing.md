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

// Assert table(s) are empty (Laravel 13.18.1+ — accepts string|iterable)
$this->assertDatabaseEmpty('posts');                  // table name (string)
$this->assertDatabaseEmpty(User::all());             // Eloquent collection
$this->assertDatabaseEmpty([$tableA, $tableB]);      // array of table names
$this->assertDatabaseEmpty(collect(['posts', 'users'])); // any iterable
```

## Validation Assertions — Use `assertInvalid` / `assertValid` (Laravel 11+)

Laravel 11 deprecated the legacy `assertJsonValidationErrors()` and `assertSessionHasErrors()` in favor of the **generic** `assertInvalid()` and `assertValid()` methods. These work uniformly for **JSON responses and session-flashed errors** — the same call covers API and web testing:

```php
// ❌ Old (Laravel 10 and earlier) — JSON-only
$response->assertJsonValidationErrors(['title', 'body']);

// ❌ Old — session-only (web form)
$response->assertSessionHasErrors(['title', 'body']);

// ✅ New (Laravel 11+) — works for BOTH JSON and session flows
$response->assertInvalid(['title', 'body']);
$response->assertValid(['title', 'body']);          // opposite: assert these did NOT error
$response->assertOnlyInvalid(['title', 'body']);    // assert ONLY these fields errored

// Assert specific error message text (substring match)
$response->assertInvalid([
    'title' => 'required',                    // error message contains "required"
    'email' => 'valid email address',         // error message contains "valid email address"
]);

// Custom error bag (e.g., default vs registration form bag)
$response->assertInvalid(['email'], errorBag: 'register');
```

**Why the new API wins:**
- One assertion method for both API (`422 JSON`) and web (`302 redirect + session`) tests
- Drop the duplicated logic of writing `assertJsonValidationErrors` for APIs and `assertSessionHasErrors` for web forms
- `assertOnlyInvalid()` catches the common bug where a validator errors on extra fields you forgot to check

## Exception Assertions — `Exceptions` Facade (Laravel 11.5+)

Stop wrapping tests in try/catch blocks to assert exceptions. Laravel's `Exceptions` facade inspects the reportable flow directly:

```php
use Illuminate\Support\Facades\Exceptions;

// Assert a specific exception was reported by the handler
Exceptions::assertReported(WelcomeException::class);
Exceptions::assertReportedCount(3);                       // reported exactly 3 times
Exceptions::assertNotReported(WelcomeException::class);   // was NOT reported (e.g., ignored)

// Assert no exceptions were reported at all (great for "happy path" tests)
Exceptions::assertNothingReported();

// Re-throw the first reported exception (useful for debugging failures)
Exceptions::throwFirstReported();
```

**Combine with `withoutExceptionHandling()` for the inverse pattern** — when you DO expect the exception to bubble up uncaught (e.g., 500 tests):

```php
public function test_missing_model_returns_500(): void
{
    $this->withoutExceptionHandling();

    try {
        $this->get('/posts/999999');
        $this->fail('Expected ModelNotFoundException was not thrown.');
    } catch (\Illuminate\Database\Eloquent\ModelNotFoundException $e) {
        $this->assertEquals(999999, $e->getModel()->id ?? null);
    }
}
```

**Pattern: assert on response status instead of the facade for negative tests.** When an exception is EXPECTED to be thrown (422 → ValidationException, 403 → AuthorizationException), the handler intercepts it — assert on the response status code:

```php
$response = $this->postJson('/api/posts', []); // missing fields
$response->assertStatus(422)->assertInvalid(['title']);
```

## Pest 3 — Architecture Testing, Mutation Testing, Arch Presets

Pest 3 ships first-class tools for catching design-level and behavioral bugs that pure unit tests miss.

```bash
composer require pestphp/pest --dev --with-all-dependencies
php artisan pest:install
composer require pestphp/pest-plugin-laravel --dev
```

### Architectural Testing — `arch()` preset

```php
// tests/Architecture/HttpTest.php
use function Pest\test;

// Default Laravel preset — controllers must be suffixed "Controller",
// have only index/show/create/store/edit/update/destroy as public methods, etc.
test('app follows Laravel conventions')
    ->arch()
    ->preset()
    ->laravel();

// Custom rules on top of the preset
test('controllers do not query the DB directly')
    ->arch()
    ->expect('App\\Http\\Controllers')
    ->not->toUse('DB::query', 'DB::select');

test('models extend Eloquent\\Model')
    ->arch()
    ->expect('App\\Models')
    ->toExtend('Illuminate\\Database\\Eloquent\\Model');

test('enums are backed strings')
    ->arch()
    ->expect('App\\Enums')
    ->toBeStringBackedEnums();

// Domain guard — production code must not import testing utilities
test('production code stays out of tests')
    ->arch()
    ->expect('App')
    ->not->toUse('Tests', 'Pest');
```

### Architectural Testing — other useful assertions

```php
test('classes are final or explicitly not')
    ->arch()
    ->expect('App\\Services')
    ->toBeFinal();   // or ->not->toBeFinal()

test('methods are typed')
    ->arch()
    ->expect('App\\Http\\Controllers')
    ->toHaveMethodsDocumented();

test('no protected methods leak')
    ->arch()
    ->expect('App\\Services')
    ->not->toHaveProtectedMethods();

test('strict equality everywhere')
    ->arch()
    ->expect('App')
    ->toUseStrictEquality();
```

### Mutation Testing — verify your tests actually catch bugs

Mutation testing injects small bugs (e.g., `>` becomes `>=`, `true` becomes `false`) and runs your test suite — if the suite still passes, you have an uncaught mutation, meaning your tests don't actually exercise that code path.

```bash
composer require pestphp/pest-plugin-mutator --dev
./vendor/bin/pest --mutate
```

Reports show **mutation score** — the % of injected bugs your suite catches. Aim for 80%+ on critical paths (controllers, services, value objects).

### Nested Describes + `after()` callback

```php
describe('PostController', function () {
    describe('store', function () {
        // Shared setup for this group only
        beforeEach(fn() => $this->user = User::factory()->create());

        after(fn() => Storage::disk('public')->deleteDirectory('uploads'));

        it('creates a post', function () { /* ... */ });
        it('validates required fields', function () { /* ... */ });
    });

    describe('destroy', function () { /* ... */ });
});
```

Source: [Pest 3 release notes](https://pestphp.com/docs/pest3-now-available) | [Pest Architecture Testing](https://pestphp.com/docs/arch-testing)

## Parallel Testing — Best Practices

`php artisan test --parallel` is a big speedup on test suites with >200 tests or heavy DB integration. Caveats:

```bash
# Run with 4 workers (default)
php artisan test --parallel

# Explicit worker count
php artisan test --parallel --processes=8

# Parallel + coverage (Laravel 11+)
php artisan test --parallel --processes=8 --coverage

# Stop after first failure (useful in CI)
php artisan test --parallel --stop-on-failure
```

**Per-worker DB isolation.** `RefreshDatabase` automatically creates per-worker test databases (e.g., `laravel_test_1`, `laravel_test_2`) when `--parallel` is on. No manual config.

**Set `APP_MAINTENANCE_DRIVER=array` in `.env.testing`** (Laravel 13.16+) when your tests call `Artisan::call('down')` / `Artisan::call('up')`. The default `file` driver shares state across workers — a `down` in worker 1 is seen by worker 2 → tests fail or interfere.

**Don't share state across tests.** If a test sets a config value, in-memory Cache::put, or singleton binding, that state will NOT be visible to other workers. Use `config()->set('key', 'value')` **inside** the test, or write to the DB, or use a per-worker cache prefix.

**Run integration tests serially if they touch external systems** (real S3 bucket, shared Redis instance). Parallel workers will collide on bucket keys / queue names.

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

// Assert a job was pushed with a specific delay (Laravel 13.18.1+)
Queue::assertPushed(ProcessPostJob::class)->delay(60);

// Assert job was NOT dispatched
Queue::assertNotPushed(ProcessPostJob::class);

// Assert job was pushed to a specific queue (Laravel 13.8+)
Queue::assertPushedOn('high-priority', ProcessPostJob::class);

// Assert job was pushed to queue defined by enum (Laravel 13.8+)
Queue::assertPushedOn(QueueNames::HighPriority, ProcessPostJob::class);

// Assert job was pushed exactly once (Laravel 13.10+)
Queue::assertPushedOnce(SendWelcomeEmailJob::class);
// Also accepts a callback for partial matching
Queue::assertPushedOnce(SendWelcomeEmailJob::class, fn($job) => $job->userId === 42);

// Batch queue assertions (Laravel 13.8+)
Queue::allPushed();                        // all dispatched jobs
Queue::allNotPushed(ProcessPostJob::class); // all jobs NOT dispatched
Queue::allPushedOn('emails');              // all jobs on 'emails' queue
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


#### assertJsonPathsCanonicalizing (Laravel 13.13+)

`assertJsonPathsCanonicalizing()` asserts multiple JSON paths where **array order doesn't matter**:

```php
$response = $this->getJson('/api/posts/1');

$response->assertJsonPathsCanonicalizing([
    'data.tags',      // array — order of tags elements doesn't matter
    'data.authors',   // array — order doesn't matter
    'data.id',        // scalar — must match
]);
```

Use this instead of `assertJsonPaths()` when the JSON contains arrays whose element order is non-deterministic (e.g., fetched from DB without ORDER BY, or merged from multiple sources).

## Session Assertions (Laravel 13.8+)

```php
// After form redirect, verify old input is NOT in session (e.g., after successful submission)
$response = $this->post('/contact', ['name' => 'Adarsh', 'email' => 'adarsh@example.com']);
$response->assertSessionMissingInput('name');  // old input cleared
$response->assertSessionMissingInput('email');

// For comparison, assert old input IS present
$response->assertSessionHasInput('name');  // redirect back with old input (validation error scenario)
```

Use `assertSessionMissingInput()` to verify that old form input has been cleared after a successful form submission (e.g., contact form sent successfully).

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
10. **Using stale old input in redirect tests** — use `assertSessionMissingInput()` after successful form submission

## Parallel Testing

Run tests in parallel processes for big speedups on test suites with many integration / DB tests:

```bash
# Run tests across 4 processes
php artisan test --parallel

# Run tests across 8 processes, then merge coverage
php artisan test --parallel --processes=8
php artisan test --parallel --processes=8 --coverage
```

Each child process gets its own DB transaction (the `RefreshDatabase` trait auto-uses a per-process DB when `--parallel` is on).

### `array` Maintenance Mode Driver (Laravel 13.16+)

The `array` maintenance mode driver (new in 13.16) is designed for **parallel test runs** that call `php artisan down` / `php artisan up`. The default `file` driver shares state across processes (a `down` in process A is seen by process B → tests fail or interfere with each other), and the `cache` driver breaks if you mock the `Cache` facade in tests.

Add this to your `.env.testing` to use the in-memory array driver:

```dotenv
APP_MAINTENANCE_DRIVER=array
```

`MaintenanceMode` then reads/writes the maintenance state from an in-memory array that's scoped to the current PHP process — clean isolation per parallel worker, no file or cache state to race on.

**When to use it:**
- Parallel test runs (`php artisan test --parallel`) that call `Artisan::call('down')` / `Artisan::call('up')` in feature tests
- CI pipelines that exercise maintenance-mode behavior without touching real files

**Caveats:**
- Driver is per-process — `down` set in one test is invisible to the next test in the same process (good for isolation; bad if you assert persistence across tests)
- Available on Laravel 13.16+ only

Source: [PR #60489](https://github.com/laravel/framework/pull/60489) | [Laravel 13.16 Release Notes](https://github.com/laravel/framework/releases/tag/v13.16.0)

## Updated from Research (2026-06-29)

### Cycle 9 additions (2026-06-29)

- **`assertInvalid()` / `assertValid()` / `assertOnlyInvalid()` (Laravel 11+)** — generic validation assertions that work for both JSON API responses and session-flashed web form errors. Replaces the split `assertJsonValidationErrors()` / `assertSessionHasErrors()` API. Supports substring matching against error messages: `$response->assertInvalid(['title' => 'required'])`.
- **`Exceptions` facade (Laravel 11.5+)** — `Exceptions::assertReported()`, `assertReportedCount()`, `assertNotReported()`, `assertNothingReported()`, `throwFirstReported()`. Replaces try/catch boilerplate for asserting exceptions were reported by the handler.
- **Pest 3 — Architecture Testing** — `test()->arch()->preset()->laravel()` enforces Laravel conventions (controller suffix, allowed public methods, etc). Plus custom rules: `->toBeFinal()`, `->toUseStrictEquality()`, `->toHaveMethodsDocumented()`, `->not->toHaveProtectedMethods()`.
- **Pest 3 — Mutation Testing** — `./vendor/bin/pest --mutate` (requires `pestphp/pest-plugin-mutator`) reports a mutation score — the % of injected bugs your test suite catches. Aim for 80%+ on critical paths.
- **Pest 3 — nested describes + `after()` callback** — group tests with `describe('group', fn() => ...)` blocks inside other describes; `after(fn() => ...)` runs cleanup scoped to that describe.

### Laravel 13.16 (June 2026)

- **`array` maintenance mode driver** — new driver for parallel-testing isolation. Pairs with `APP_MAINTENANCE_DRIVER=array` in `.env.testing` (see section above).

### Laravel 13.10 (May 2026)

- **assertPushedOnce()** — `Queue::assertPushedOnce($job, $callback?)` asserts a job was pushed exactly once. Equivalent to `assertPushed` with exactly 1 expected call, but cleaner and more explicit. Optionally accepts a callback for partial matching.

### Laravel 13.8 (May 2026)

- **assertSessionMissingInput** — verify old form input is cleared from session after successful submission
- **QueueFake batch methods** — `allPushed()`, `allNotPushed()`, `allPushedOn()` for batch job assertions
- **QueueFake assertPushedOn enum** — `assertPushedOn()` accepts an enum as the queue name

### Laravel 13.7+ Testing

- **Bulk JSON Path Assertions** — `assertJsonPaths()` for asserting multiple JSON paths in one call.

### Laravel 13 Testing Attributes

- **#[Group], #[TestProperty], #[UnitTest]** — organize tests, attach metadata, skip app boot for pure unit tests.

### General Testing

- **Pest PHP** — increasingly standard in Laravel ecosystem; `pest()` function replaces `TestCase` class methods.
- **HTTP Client Mocking** — `Http::fake()` for mocking external API calls in tests.
- **Mocking Best Practices** — use `mock()` for services, `spy()` when you only need to verify calls happened, `Queue::fake()` for job queues.
- **`assertDatabaseEmpty()` Accepts Iterables (Laravel 13.18.1, PR #60621 by @jackbayliss)** — previously only worked with table-name strings (`'posts'`); passing an Eloquent collection or Builder threw `Argument #1 must be of type string`. Now `assertDatabaseEmpty(string|iterable $table)` accepts collections, arrays of table names, or any iterable. Lets you write `$this->assertDatabaseEmpty(User::all())` to verify a table is empty after a destructive operation.

Source: [Laravel 13 Docs - Testing](https://laravel.com/docs/13.x/testing)
