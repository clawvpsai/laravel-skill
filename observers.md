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

## Laravel 13 Attribute Events (New)

Laravel 13 formalizes model attribute events with PHP attributes:

```php
use App\Models\Post;
use Illuminate\Database\Eloquent\Attributes\UpdatedAt;

class Post extends Model
{
    // Fires on any attribute update
    #[UpdatedAt]
    public function touch(): void
    {
        // Custom logic on any update
    }
}
```

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

## Updated from Research (2026-05)
- Laravel 13 introduces formal PHP attributes for model event hooks
- `ShouldQueue` on listeners is the recommended pattern for async event processing

Source: [Laravel Events](https://laravel.com/docs/13.x/events)
