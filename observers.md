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
        // Restored from soft delete
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
        // Optionally restore related records
    }
}
```

## Event Best Practices

1. **Keep events simple** — only pass necessary data, not full models
2. **Make events serializable** — don't pass unserializable objects (file handles, DB connections)
3. **Use queues for slow operations** — email, notifications, external API calls in `ShouldQueue` listeners
4. **One listener per responsibility** — don't cram everything into one observer
5. **Don't cause infinite loops** — updating model in its own observer that fires another update

## Common Mistakes

1. **Observer updating the model it observes** — use `$post->updateQuietly()` to avoid recursion
2. **Events firing on mass updates** — `Model::query()->update()` bypasses observers; use chunked individual updates
3. **Not using queue for slow listeners** — blocking the request with emails/notifications
4. **Storing unserializable data in events** — events get queued; file handles, connections can't serialize
5. **Overusing events** — for simple cases, just put logic directly in controller or service

## Updated from Research (2026-05-14)

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
