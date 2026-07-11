# Observers & Events — Model Lifecycle & Decoupled Logic

## Observers (Model Event Listeners)

```bash
php artisan make:observer PostObserver --model=Post
```

```php
namespace App\Observers;

use App\Models\Post;

class PostObserver
{
    public function created(Post $post): void
    {
        // New post created
    }

    public function updated(Post $post): void
    {
        // Post updated — check $post->getChanges() for dirty fields
        if ($post->wasChanged('published_at') && $post->published_at) {
            // Post just went live
        }
    }

    public function deleted(Post $post): void
    {
        // Soft delete also triggers this
    }

    public function forceDeleted(Post $post): void
    {
        // Permanent deletion
    }

    public function restored(Post $post): void
    {
        // Restored from soft delete.
        //
        // Laravel 13.17+ (PR #60605): the `restored` event fires ONLY when the
        // underlying save() succeeded. If a `saving` listener returned `false`
        // (or the DB rejected the row), the event will NOT fire — check
        // `restore()`'s boolean return before assuming restoration happened.
    }

    public function retrieved(Post $post): void
    {
        // Model loaded from DB
    }
}
```

**Register in service provider:**
```php
// AppServiceProvider boot()
use App\Models\Post;
use App\Observers\PostObserver;

Post::observe(PostObserver::class);
```

## Laravel 13 `#[ObservedBy]` Attribute (No Boot Method Needed)

Instead of registering observers in a service provider's `boot()` method, Laravel 13 lets you attach observers directly via the `#[ObservedBy]` PHP attribute on the model — cleaner and co-locates the observer with the model:

```php
use App\Models\Post;
use App\Observers\PostObserver;
use Illuminate\Database\Eloquent\Attributes\ObservedBy;

#[ObservedBy([PostObserver::class])]
class Post extends Model
{
    // Observer is auto-attached — no service provider needed
}
```

**With multiple observers:**
```php
#[ObservedBy([PostObserver::class, PostAnalyticsObserver::class])]
class Post extends Model
{
    // Both observers receive all model events
}
```

**Benefits over boot method:**
- Observer is visible right next to the model it concerns
- No service provider pollution
- Easier to see at a glance what model lifecycle hooks exist
- Works well with IDE autocompletion for the attribute

**Legacy boot method still works:**
```php
// Still valid in Laravel 13 — not going away
protected static function boot()
{
    parent::boot();
    static::observe(PostObserver::class);
}
```

## Events (Decoupled Architecture)

```php
// Define event
php artisan make:event PostPublished

class PostPublished
{
    public function __construct(public Post $post, public ?User $publishedBy) {}
}

// Fire
event(new PostPublished($post, auth()->user()));

// Named event
event(new PostPublished($post, auth()->user()));
```

**Listener:**
```php
php artisan make:listener SendPostPublishedNotification --event=PostPublished

class SendPostPublishedNotification
{
    public function handle(PostPublished $event): void
    {
        Mail::to($event->post->author)->send(new PostLiveMail($event->post));
    }
}
```

**Register in `EventServiceProvider` or `bootstrap/app.php`:**
```php
// Laravel 11+
use App\Events\PostPublished;
use App\Listeners\SendPostPublishedNotification;

Event::listen(PostPublished::class, SendPostPublishedNotification::class);
```

## Events as Queue Jobs (Async)

```php
// Auto-dispatched to queue — just implement ShouldQueue
class SendPostPublishedNotification implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable;

    public function handle(PostPublished $event): void
    {
        Mail::to($event->post->author)->send(new PostLiveMail($event->post));
    }
}
```

## Laravel 13 Model Attribute Hooks

Laravel 13 introduces PHP attributes for model lifecycle hooks directly on model methods:

```php
use App\Models\Post;
use Illuminate\Database\Eloquent\Attributes\UpdatedAt;

class Post extends Model
{
    // Called when the updated_at timestamp is set (e.g., via touch() or save)
    #[UpdatedAt]
    public function onTimestampUpdate(): void
    {
        // Custom logic — e.g., notify subscribers when post is edited
    }
}
```

**Available lifecycle attributes:**
- `#[UpdatedAt]` — method fires when `updated_at` is set

These complement (not replace) observer methods. Use for lightweight inline hooks; use observers for heavier logic that belongs in its own class.

## Model Custom Events (Beyond Observer)

```php
// Instead of observer, use model events with custom event classes
class Post extends Model
{
    protected $dispatchesEvents = [
        'created' => PostCreated::class,
        'updated' => PostUpdated::class,
        'deleted' => PostDeleted::class,
    ];
}
```

## Observer for File Cleanup

```php
class PostObserver
{
    public function deleted(Post $post): void
    {
        // Delete associated image
        Storage::disk('public')->delete($post->featured_image);
        
        // Delete related comments
        $post->comments()->delete();
    }

    public function restored(Post $post): void
    {
        // Optionally restore related records.
        //
        // Laravel 13.17+ (PR #60605): this only fires when the underlying
        // save() succeeded. If a `saving` listener returned false, this event
        // is skipped — check the boolean return of `$model->restore()` instead.
    }
}
```

## Event Best Practices

1. **Keep events simple** — only pass necessary data, not full models
2. **Make events serializable** — don't pass unserializable objects (file handles, DB connections)
3. **Use queues for slow operations** — email, notifications, external API calls in `ShouldQueue` listeners
4. **One listener per responsibility** — don't cram everything into one observer
5. **Don't cause infinite loops** — updating model in its own observer that fires another update


## `ShouldQueue` on Observers (Different Mechanics than Listener `ShouldQueue`)

Observers can implement `ShouldQueue` directly — and the behavior differs subtly from listeners that implement it. Important details:

```php
use App\Models\Order;
use Illuminate\Contracts\Queue\ShouldQueue;

class OrderObserver implements ShouldQueue
{
    public function created(Order $order): void
    {
        // Runs in queue worker, NOT in the request that created the order
        ProcessOrderAnalytics::dispatch($order);
        UpdateRecommendations::dispatch($order->customer);
    }
}
```

**What gets serialized to the queue:**
- The observer class name
- The event method name (`created`, `updated`, etc.)
- The Eloquent model's primary key (NOT the full model) — the queue worker re-fetches it from the DB via `Model::find($id)`
- Any constructor arguments you pass to the observer (must be queue-serializable)

**Critical gotcha — model re-fetch happens after the request returns.** If you delete the model inside the same request after the observer is queued, the queue worker will fail to find it on re-fetch and the job will throw `ModelNotFoundException`. Solutions:
- Pass the data the observer needs as a plain DTO in the observer constructor (`new OrderObserver($order->id, $order->total)`)
- Or wrap the model in an event class and dispatch the event with `ShouldQueue` listeners that pull only what they need
- Or schedule the observer to run synchronously by NOT implementing `ShouldQueue`

**`ShouldQueue` + retry behavior on observers:** by default `tries = 1`. To retry:

```php
class OrderObserver implements ShouldQueue
{
    public int $tries = 3;
    public int $backoff = 60; // seconds between retries
    public int $maxExceptions = 2; // only retry on non-fatal exceptions
}
```

Same attributes work on observers as on jobs (`#[Job\Backoff(60)]`, `#[Job\MaxAttempts(3)]`, `#[Job\Timeout(120)]` for Laravel 13).

**When NOT to make an observer `ShouldQueue`:**
- The observer does critical synchronous work (audit log writes that must exist before the request returns)
- The observer has side effects on the response (e.g., fires a `flash()` message — meaningless after the request returns)
- The observer is in a tightly-coupled write path where the queued re-fetch latency is unacceptable

Source: [Laravel Docs - Queued Observers](https://laravel.com/docs/13.x/eloquent#observers-and-database-transactions)

## `ShouldHandleEventsAfterCommit` — Transaction-Safe Observers

When an observer fires inside a `DB::transaction()` (explicit or implicit via `Model::save()`), the work happens *during* the transaction. If the transaction rolls back, the observer has already done its work — emails sent, webhooks fired, cache invalidated — but the data wasn't actually persisted. **Data inconsistency.**

```php
use Illuminate\Contracts\Events\ShouldHandleEventsAfterCommit;

class SendOrderConfirmation implements ShouldQueue, ShouldHandleEventsAfterCommit
{
    public function handle(OrderShipped $event): void
    {
        Mail::to($event->order->customer)->send(new OrderShippedMail($event->order));
    }
}
```

With `ShouldHandleEventsAfterCommit`:
- For **queued listeners**: the job is dispatched only after the outer transaction commits. If the transaction rolls back, the job is never dispatched.
- For **sync listeners**: Laravel defers dispatch until `DB::afterCommit()` would fire — same end result.

**Known gotcha (GitHub issue [#52440](https://github.com/laravel/framework/issues/52440)):** if a listener implements BOTH `ShouldQueue` AND `ShouldHandleEventsAfterCommit`, the `ShouldQueue` early-return in `Event/Dispatcher.php` skips the `ShouldHandleEventsAfterCommit` check on Redis/SQS queue drivers. Workaround: pick one — for most cases `ShouldQueue` is enough because the queued job only fires when the worker pulls it, which is after commit if the worker dispatches the job through `DB::afterCommit()` (Laravel does this automatically in 11+).

**Same pattern works on observers:**

```php
class OrderObserver implements ShouldQueue, ShouldHandleEventsAfterCommit
{
    public function created(Order $order): void { /* fires AFTER transaction commits */ }
}
```

**Without `ShouldHandleEventsAfterCommit`**, your observer's side effects leak into rolled-back transactions. Always pair queued observers/listeners with `ShouldHandleEventsAfterCommit` if they're inside a `DB::transaction()` block.

## Octane & Static Observer Registration — State Leak Between Requests

Under Octane (or RoadRunner / FrankenPHP-octane), the PHP process is reused across requests. `#[ObservedBy]` and `static::observe()` register observers **statically** on the model class — they survive between requests. Normally this is fine (you want the observer registered once). But:

- If your observer's constructor reads request-scoped state (request ID, tenant, auth user), that state **leaks** to subsequent requests in the same worker.
- If your observer registers dynamically inside `boot()` based on config that changes between requests, the config snapshot from request 1 persists into request 5.

```php
// BAD — request-scoped state in observer constructor under Octane
class TenantAwareObserver
{
    public function __construct(public Tenant $tenant) {}

    public function created(Post $post): void
    {
        // $this->tenant is from the request that first booted this observer,
        // NOT the request that just created $post
    }
}
```

**Solutions:**
- **Inject via method args, not constructor** — read `app('current_tenant')` inside the method body, not in `__construct`.
- **Don't implement `ShouldQueue`** — queued observers are reconstructed by the queue worker each time, so request-scoped state is fresh.
- **Reset state at the end of each request** — use Octane's `Octane::flush()` callback in your service provider to call `Post::flushEventListeners()` for any observers you've registered dynamically.

```php
// app/Providers/AppServiceProvider.php
public function boot(): void
{
    \Laravel\Octane\Octane::flush(function () {
        // Reset observers registered dynamically per request
        Post::flushEventListeners();
    });
}
```

Source: [Laravel Octane Docs - Dependency Injection & Octane](https://laravel.com/docs/13.x/octane#dependency-injection-and-octane)

## Quiet Patterns — `withoutEvents()` vs `updateQuietly()` vs `saveQuietly()`

Three ways to update a model without firing observers. Each has different scope:

| Method | Fires observer events? | Touches `updated_at`? | Triggers model events globally? | Use for |
|---|---|---|---|---|
| `$model->update([...])` | Yes | Yes | Yes (for this model) | Normal updates |
| `$model->updateQuietly([...])` | No | Yes | No (this model only) | Self-referential observer updates — preventing infinite loops |
| `$model->saveQuietly([...])` | No | Yes | No (this model only) | Same as `updateQuietly` but with full save() semantics (events, casts, etc. suppressed) |
| `Model::withoutEvents(fn () => Model::query()->update([...]))` | No | Yes (if `$timestamps = true`) | No (ALL models in the closure) | Mass updates, seeder bulk operations, migrations |
| `Model::withoutEvents(function () { ... })` | No | Yes | No (ALL models in the closure) | Same as above but for arbitrary work |

**Practical example — observer updating the model it observes:**

```php
class PostObserver
{
    public function updated(Post $post): void
    {
        // BAD — infinite loop:
        // $post->update(['last_indexed_at' => now()]);

        // GOOD — quietly set the flag without re-firing `updated`:
        $post->updateQuietly(['last_indexed_at' => now()]);
    }
}
```

**Practical example — seeder doing mass update:**

```php
// BAD — fires `updated` on every post (slow, side effects)
Post::query()->where('legacy', true)->update(['migrated' => true]);

// GOOD — no observer side effects
Post::withoutEvents(fn () =>
    Post::query()->where('legacy', true)->update(['migrated' => true])
);
```

**Practical example — flush-then-import:**

```php
// Inside a migration or seeder:
Post::withoutEvents(function () {
    Post::query()->truncate(); // no `deleted` events fire
    Post::insert($rows);       // no `created` events fire
});
```

**Gotcha:** `updateQuietly()` / `saveQuietly()` only suppress events for the model instance you call them on. They do NOT suppress events for related models you touch inside the observer. If your `updated` observer calls `$post->comments()->save(...)`, the comment save still fires its own events — wrap that in `Comment::withoutEvents(...)` too if needed.

## Testing Observers — `Event::fake()` vs `Event::fakeFor()`

Two patterns for testing observer-driven code, with different scope:

```php
// Event::fake() — swaps the dispatcher for the rest of the test
public function test_post_creation_sends_notification(): void
{
    Event::fake();

    $post = Post::create([...]);

    Event::assertDispatched(PostCreated::class);
    Event::assertNotDispatched(PostDeleted::class);
}
```

```php
// Event::fakeFor() — swaps the dispatcher only inside the closure
public function test_index_endpoint_doesnt_fire_observer(): void
{
    $fired = Event::fakeFor(function () {
        return Post::create([...]);
    });

    $fired->assertDispatched(PostCreated::class);
}
```

**For observers specifically:**
- `Event::fake([PostCreated::class])` — only fake the events you care about; OTHER events still fire normally. This is the right pattern when your observer also dispatches downstream events (e.g., `PostCreated` fires `SearchIndexUpdated`) and you want to assert on the primary without losing the rest.
- `Event::fake()` (no args) — fakes ALL events. Observer side effects don't run. Use this when you want pure unit-test isolation and the observer's side effects are tested elsewhere.
- `Event::fakeFor()` — same as `Event::fake()` but scoped to a closure. Useful inside a single test method when you only want to fake one specific call.

**Don't fake events when the test depends on the observer's side effect.** If you're testing that creating a post populates the search index, don't fake `SearchIndexUpdated` — let it fire (with a fake search client).

**For queued observers (`ShouldQueue`):** use `Bus::fake()` instead of `Event::fake()`. The event dispatches synchronously (because the listener is the observer itself, not a queued job), but if your observer dispatches further queued jobs internally, `Bus::fake()` is what catches those.

## Selective Event Firing — `setObservableEvents()`

By default, Eloquent fires ALL model events. You can narrow this down per-instance or per-class:

```php
// Disable specific events on one model instance
$post = Post::find(1);
$post->setObservableEvents(['created', 'updated']); // ONLY these fire
$post->save(); // fires 'created' or 'updated', nothing else

// Reset to defaults
$post->setObservableEvents(array_keys($post->getObservableEvents()));
```

**Common use cases:**
- Read-only DTO models that only fire `retrieved` (and skip `saving`/`saved` etc.)
- Sync import scripts that want `created` but not `updated` (so accidental re-imports don't fire `updated`)
- Audit-log-only observers attached to a model that suppresses `creating`/`created` (only logs deletes and updates)

**Be careful:** `setObservableEvents()` is instance-scoped by default. For class-wide filtering, override `getObservableEvents()` in the model:

```php
class AuditOnlyPost extends Post
{
    public function getObservableEvents(): array
    {
        return ['updated', 'deleted']; // never fires created, saving, saved, etc.
    }
}
```

This is also useful for performance — observers with expensive side effects can be filtered out per-deployment.


## Common Mistakes

1. **Observer updating the model it observes** — use `$post->updateQuietly()` to avoid recursion
2. **Events firing on mass updates** — `Model::query()->update()` bypasses observers; use chunked individual updates
3. **Not using queue for slow listeners** — blocking the request with emails/notifications
4. **Storing unserializable data in events** — events get queued; file handles, connections can't serialize
5. **Overusing events** — for simple cases, just put logic directly in controller or service
6. **Queued observer inside `DB::transaction()` without `ShouldHandleEventsAfterCommit`** — side effects fire even if the transaction rolls back; data inconsistency
7. **Assuming `updateQuietly()` suppresses events on related models** — it only suppresses for the instance it's called on; wrap related-model touches in `RelatedModel::withoutEvents()` too
8. **Putting request-scoped state (auth user, tenant, request ID) in observer constructor under Octane** — state from request 1 leaks into request 5+; inject via method body instead
9. **Using `Event::fake()` when the test depends on observer side effects** — fake only the events you don't care about (`Event::fake([OtherEvent::class])`)
10. **Queueing observers with critical synchronous side effects (audit logs, flash messages)** — queue work runs after the request returns; sync observer for work that must complete before response
11. **Forgetting that queued observers re-fetch the model by primary key** — if the model is deleted in the same request, the queue worker gets `ModelNotFoundException`; pass the data as a DTO or via constructor args instead


## Laravel `#[Boot]` and `#[Initialize]` Model Attributes (Pre-Laravel 13, missing from older docs)

These PHP attributes let you hang lifecycle logic off individual model methods instead of overriding `boot()` or wrapping everything in an observer:

```php
use App\Models\Post;
use Illuminate\Database\Eloquent\Attributes\Boot;
use Illuminate\Database\Eloquent\Attributes\Initialize;
use Illuminate\Support\Str;

class Post extends Model
{
    // #[Boot] runs ONCE when the model class is first booted — like static::created() inside boot()
    #[Boot]
    public static function registerEvents(): void
    {
        static::creating(fn (Post $post) => $post->slug = Str::slug($post->title));
        static::updating(fn (Post $post) => Cache::forget("post:{$post->getKey()}"));
    }

    // #[Initialize] runs EVERY time a new instance is hydrated — like a constructor hook without overriding __construct()
    #[Initialize]
    public function setDefaults(): void
    {
        $this->attributes['status'] ??= 'draft';
        $this->attributes['uuid'] ??= (string) Str::uuid();
    }
}
```

**When to use what:**
- `#[Boot]` — register **static** event listeners, global scopes, or anything that should only happen once per class lifecycle. Replaces the body of `protected static function boot()`.
- `#[Initialize]` — set per-instance defaults that aren't covered by `$attributes` defaults (e.g., randomized UUIDs, computed initial state). Runs after `__construct()` and after the database has hydrated attributes.
- Observer class — when logic is heavy or shared across many models. `#[ObservedBy]` is still preferred for cross-cutting concerns (audit logs, cache invalidation, search index sync).

**`#[Scope]` (related attribute):**
```php
use Illuminate\Database\Eloquent\Attributes\Scope;
use Illuminate\Database\Eloquent\Builder;

class Post extends Model
{
    #[Scope]
    public function published(Builder $query): void
    {
        $query->whereNotNull('published_at');
    }
}
// Usage: Post::query()->published()->get();
```

**Pair with `#[ObservedBy]`** — `#[Boot]` handles in-model event listeners; `#[ObservedBy]` attaches a full observer class. They compose cleanly on the same model.

## `ShouldBeDiscovered` Opt-Out (Laravel 13.12+)

When you use Laravel's **automatic event discovery** (`Event::shouldDiscoverEvents()` in `bootstrap/app.php`), every listener that implements `ShouldQueue` is auto-registered as a queued job — and any matching event dispatch will queue it. The 13.12 release adds an opt-out for listeners you don't want discovered at all, even when discovery is on:

```php
use Illuminate\Contracts\Queue\ShouldQueue;

// This listener is registered via Event::listen() in bootstrap/app.php ONLY.
// It will NOT be picked up by Event::shouldDiscoverEvents() discovery.
class SendUrgentAdminAlert implements ShouldQueue
{
    public static bool $shouldDiscover = false; // ← opt-out flag

    public function handle(InvoicePaid $event): void
    {
        // ...
    }
}
```

The exact opt-out contract name varies between 13.12 patch levels; the canonical mechanism is a static `$shouldDiscover = false` property OR a marker interface under `Illuminate\Contracts\Events\` (verify the contract name against your installed version's `vendor/laravel/framework/src/Illuminate/Contracts/Events/` directory before relying on it). Either way, the semantic is the same: **this listener is never auto-registered, you must register it manually** with `Event::listen()` in `bootstrap/app.php`.

**Why this matters:**
- **Performance** — if a queued listener should only run on a specific event class that's hard-coded in your app (e.g., a webhook for one Stripe product), you don't want discovery to register it for *every* `OrderShipped` event.
- **Correctness** — listeners that have side effects you want to control (audit logs, billing side-effects) shouldn't fire via discovery; they should fire via your explicit `Event::listen()` call only.
- **Dead-job prevention** — without the opt-out, a `ShouldQueue` listener sits in the discovered-listener registry even if the event never fires in your app, inflating queue worker boot time and confusing `queue:monitor`.

**When to use:**
- Discovery is enabled globally (`Event::shouldDiscoverEvents()`) but you have a few listeners you want to keep explicit.
- A `ShouldQueue` listener is only meaningful for one specific dispatch site (e.g., `event(new InvoicePaid($invoice))` from a billing webhook) and you don't want it firing on every other event it might match.
- A listener has destructive side effects (database writes, third-party API calls) and you want a code review to gate its registration.

**When NOT to use:**
- You want the listener to be auto-discovered (default behavior).
- The listener is sync (`handle()` without `ShouldQueue`) — discovery of sync listeners doesn't queue anything, so there's nothing to opt out of.

Source: [Laravel News — Scheduler Attributes & Listener Discovery Control in 13.12.0](https://laravel-news.com/laravel-13-12-0) | [Laravel 13.12 Release Notes](https://github.com/laravel/framework/releases/tag/v13.12.0)

## Updated from Research (2026-06-29)

- **Soft-delete `restore()` event gating (PR #60605, 13.17+)** — the `restored` model event now only fires when the underlying `save()` actually returned true. Before 13.17 it fired unconditionally, which made `static::restored(fn() => $x = true)`-based tests report success even when a `saving` listener had cancelled the restore. Always check the return value of `$model->restore()` (boolean) instead of relying on the event firing.


### `#[ObservedBy]` Attribute (Laravel 13)

- **No service provider needed** — attach observer directly to model via `#[ObservedBy([PostObserver::class])]`
- Attaches observer automatically when model is loaded
- Supports multiple observers: `#[ObservedBy([Obs1::class, Obs2::class])]`
- Legacy `static::boot()` + `static::observe()` pattern still works

### Model Lifecycle Attribute Hooks (Laravel 13)

- `#[UpdatedAt]` on a method marks it as the handler when the `updated_at` timestamp is set
- Lightweight alternative to full observer for simple inline hooks

### General

- `ShouldQueue` on listeners is the recommended pattern for async event processing
- Prefer `#[ObservedBy]` over service provider observer registration for new code

Sources: [Laravel 13 Docs - Eloquent Observers](https://laravel.com/docs/13.x/eloquent#observers) | [Laravel 13 Docs - Events](https://laravel.com/docs/13.x/events) | [Laravel News - PHP Attributes](https://laravel-news.com/laravel-13-attributes)


### `#[Boot]` and `#[Initialize]` Model Attributes

- **`#[Boot]`** — PHP attribute on a static method, runs once when the model class is first booted. Replaces the body of `static function boot()`. Use for event listeners, global scopes, anything that should fire once per class lifecycle.
- **`#[Initialize]`** — PHP attribute on an instance method, runs every time a new instance is hydrated (after `__construct()` and DB hydration). Use for randomized UUIDs, computed defaults, anything `$attributes` defaults can't cover.
- **`#[Scope]`** — attribute on a query-scope method, lets you write `#[Scope] public function published(Builder $query): void` without the `scope` prefix.
- All three shipped pre-Laravel 13 but were missing from this skill.
- **`ShouldBeDiscovered` opt-out (Laravel 13.12+)** — opt-out mechanism for automatic event discovery. Use for `ShouldQueue` listeners with narrow dispatch sites or destructive side effects that you want registered explicitly via `Event::listen()`, never via `Event::shouldDiscoverEvents()`.
