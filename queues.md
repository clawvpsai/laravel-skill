# Queues — Jobs, Workers, Retries

## The Critical Warning

**Queue jobs serialize models as IDs, not the full object.**

```php
class ProcessPostJob implements ShouldQueue
{
    public function __construct(public Post $post) {} // serializes post_id

    public function handle()
    {
        // $this->post is re-fetched from DB — might be deleted or stale
        // Always re-fetch if you need current data
        $post = Post::find($this->post->id);
    }
}
```

**Solution:** Re-fetch models inside `handle()` or pass IDs and fetch inside.

## Job Setup

```bash
# Create job
php artisan make:job ProcessPostJob

# Failed jobs table
php artisan queue:failed-table
php artisan migrate
```

## Job Structure

```php
<?php

namespace App\Jobs;

use App\Models\Post;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;

class ProcessPostJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;        // retry up to 3 times
    public int $backoff = 60;     // wait 60s before retry (override per-failure with $this->backoff)
    public int $timeout = 120;    // job fails if not done in 2 min
    public int $maxExceptions = 2; // fail permanently after 2 unhandled exceptions

    public function __construct(public int $postId) {}

    public function handle(): void
    {
        $post = Post::find($this->postId);
        if (!$post) {
            Log::warning("Post {$this->postId} not found during processing");
            return; // don't retry if post was deleted
        }

        // Do work
        $post->update(['processed' => true]);
    }

    public function failed(\Throwable $e): void
    {
        Log::error("ProcessPostJob failed permanently", [
            'post_id' => $this->postId,
            'error' => $e->getMessage(),
        ]);
    }

    // Per-attempt backoff
    public function backoff(): array
    {
        return [60, 300, 900]; // 1min, 5min, 15min between retries
    }

    // Specify which queue
    public function queue(): string
    {
        return 'processing'; // jobs go to 'processing' queue
    }
}
```

## Laravel 13 PHP Attributes (Alternative to Properties)

Instead of defining job properties, you can use PHP attributes:

```php
use Illuminate\Queue\Attributes\Job;

#[Job]
#[Job\Backoff([60, 300, 900])]
#[Job\MaxAttempts(3)]
#[Job\Timeout(120)]
#[Job\FailOnTimeout]
class ProcessPostJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(public int $postId) {}

    public function handle(): void
    {
        $post = Post::find($this->postId);
        if (!$post) return;
        $post->update(['processed' => true]);
    }

    public function failed(\Throwable $e): void
    {
        Log::error("ProcessPostJob failed", ['post_id' => $this->postId, 'error' => $e->getMessage()]);
    }
}
```

## Laravel 13.7+ Interruptible Jobs (ShouldInterrupt)

Jobs can implement `ShouldInterrupt` to respond to worker shutdown signals — useful for long-running jobs that need to checkpoint progress:

```php
use Illuminate\Contracts\Queue\ShouldInterrupt;
use Illuminate\Queue\InteractsWithQueue;

class ProcessLargeDatasetJob implements ShouldQueue, ShouldInterrupt
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 1;  // no retries — handle gracefully on interrupt
    public int $timeout = 3600;

    public function __construct(public int $datasetId) {}

    public function handle(): void
    {
        $items = DatasetItem::where('dataset_id', $this->datasetId)->cursor();

        foreach ($items as $item) {
            // Checkpoint — save progress before potentially being interrupted
            cache()->put("dataset:{$this->datasetId}:checkpoint", $item->id);

            // Process item
            $this->processItem($item);

            // Laravel checks interruption at this point (yielded via cursor iteration)
        }
    }

    public function interrupt(): void
    {
        // Save final checkpoint — job will resume here on next run
        Log::info("Job interrupted, checkpoint saved", [
            'dataset_id' => $this->datasetId,
            'last_item_id' => cache()->get("dataset:{$this->datasetId}:checkpoint"),
        ]);
    }
}
```

**When to use:**
- Long-running batch jobs (data imports, report generation)
- Jobs that can resume from a checkpoint rather than restart from scratch
- Worker graceful shutdown (SIGTERM) triggers interruption point



## Debounceable Jobs (Laravel 13.6+)

When the same job is dispatched multiple times in a short window, debounceable jobs keep **only the last one** — eliminating redundant processing from bursty user actions. Apply `#[DebounceFor]` on the job class or debounce at the dispatch call site:

```php
use Illuminate\Queue\Attributes\DebounceFor;
use Illuminate\Queue\Attributes\Job;

#[Job]
#[DebounceFor(30)] // only keep the last dispatch if another comes within 30 seconds
class RebuildDocumentJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(public int $documentId) {}

    public function handle(): void
    {
        // Rebuild document — only runs once even if dispatched 10 times in 30s
        Document::find($this->documentId)?->rebuild();
    }
}
```

**Debounce at dispatch time** (overrides the class-level attribute):
```php
RebuildDocumentJob::dispatch($docId)->debounce(60);
```

**When to use:**
- Auto-save / document rebuilding where user edits rapidly
- Search index updates after multiple changes
- Notifications that should only fire once after a burst of activity
- Rate-limiting expensive jobs without external throttle infrastructure

## Dispatching

```php
// Fire and forget (default)
ProcessPostJob::dispatch($postId);

// Delay
ProcessPostJob::dispatch($postId)->delay(now()->addMinutes(5));

// Chain (sequential)
ProcessPostJob::dispatch($postId)
    ->chain([new SendPostNotificationJob($postId)]);

// On specific queue
ProcessPostJob::dispatch($postId)->onQueue('high-priority');

// Sync (no queue — runs immediately, good for dev/debug)
ProcessPostJob::dispatchSync($postId);
```

### PendingDispatch Conditionals (Laravel 13.9+)

`PendingDispatch` now supports `->when()` and `->unless()` for clean conditional dispatch:

```php
// Dispatch only if condition is true
PendingDispatch::dispatch($postId)
    ->when($user->isPremium(), fn($job) => $job->onQueue('premium'));

// Or at dispatch time with the helper
dispatch(new ProcessPostJob($postId))
    ->when($isPriority)
    ->delay(now()->addMinutes(5));
```

**Use case:** Skip expensive jobs if a feature flag is off, or route premium users to a separate queue.

```php
// Route premium users to dedicated queue via condition
dispatch(new SendMarketingEmailJob($userId))
    ->when(
        $user->subscription->plan === 'premium',
        fn($job) => $job->onQueue('marketing-premium')
    );
```


## Queue Workers

```bash
# Run default queue
php artisan queue:work

# Run specific queue (process high-priority first)
php artisan queue:work --queue=high-priority,default

# Listen (auto-restarts on code changes — better for dev)
php artisan queue:listen

# Run single job (for testing)
php artisan queue:work --once

# Process all pending jobs then exit
php artisan queue:work --stop-when-empty
```

**Supervisor config for production (`/etc/supervisor/conf.d/laravel-worker.conf`):**
```ini
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /path/to/artisan queue:work redis --queue=high-priority,default --tries=3 --backoff=60
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
numprocs=4
redirect_stderr=true
stdout_logfile=/var/log/laravel-worker.log
stopwaitsecs=3600
```

## Queue Routing (Laravel 13+)

Define default queue/connection routing rules for specific jobs centrally:

```php
// bootstrap/app.php or AppServiceProvider
use Illuminate\Support\Facades\Queue;
use App\Jobs\ProcessPodcast;

Queue::route(ProcessPodcast::class, connection: 'redis', queue: 'podcasts');

// Now all ProcessPodcast dispatches go to redis podcasts queue automatically
ProcessPodcast::dispatch($podcastId); // -> redis:podcasts
```

## Worker Lifecycle Events (Laravel 13.8+)

Laravel 13.8 adds `WorkerPausing` and `WorkerResuming` events dispatched by queue workers — useful for coordinating external resources (database connections, Redis sessions, file handles) when workers pause or resume:

```php
use Illuminate\Queue\Events\WorkerPausing;
use Illuminate\Queue\Events\WorkerResuming;
use Illuminate\Support\Facades\Event;

// In AppServiceProvider boot()
Event::listen(\Illuminate\Queue\Events\WorkerPausing::class, function (WorkerPausing $event) {
    // Worker is about to pause (e.g., queue empty, SIGSTOP, scaling down)
    // Close or release non-persistent resources
    \Illuminate\Support\Facades\DB::disconnect();
    Log::info("Worker pausing", ['connection' => $event->connection, 'queue' => $event->queue]);
});

Event::listen(\Illuminate\Queue\Events\WorkerResuming::class, function (WorkerResuming $event) {
    // Worker is resuming after a pause
    // Reconnect resources, warm up caches
    Log::info("Worker resuming", ['connection' => $event->connection, 'queue' => $event->queue]);
});
```

**Use cases:**
- Release database connection pools during idle pauses to free resources
- Flush or sync local caches when workers scale down/up
- Coordinate with Kubernetes pod lifecycle or AWS auto-scaling graceful shutdown
- Log worker availability state for monitoring dashboards

## Failed Jobs

```bash
# List
php artisan queue:failed

# Retry single
php artisan queue:retry 5

# Retry all
php artisan queue:retry --all

# Forget (delete)
php artisan queue:forget 5

# Flush (delete all)
php artisan queue:flush
```

## Job Middleware

```php
// Rate limiting
class RateJobMiddleware
{
    public function handle($job, $next)
    {
        if (RateLimiter::tooManyAttempts('job:' . $job->attempts(), 3)) {
            return $job->release(60); // retry in 60s
        }
        $next($job);
    }
}

// Register in AppServiceProvider
Queue::beforeWorking(function (Job $job) {
    Log::info("Starting job", ['job' => get_class($job)]);
});
```

## Batching

```php
use Illuminate\Bus\Batch;

$batch = Bus::batch([
    new ProcessPostJob($post->id),
    new ProcessPostJob($post2->id),
    new ProcessPostJob($post3->id),
])->then(function (Batch $batch) {
    // All jobs completed successfully
    Log::info("Batch done", ['total' => $batch->total()]);
})->catch(function (Batch $batch, \Throwable $e) {
    // Any job failed
    Log::error("Batch failed", ['error' => $e->getMessage()]);
})->finally(function (Batch $batch) {
    // Always runs
})->dispatch();

// Check batch progress
$batch->progress(); // 0-100
$batch->failed(); // number of failures
```

## Common Mistakes

1. **Passing Eloquent models to jobs** — models serialize as IDs, re-fetch inside handle()
2. **Not setting timeout** — job runs forever and holds worker hostage
3. **No failed() handler** — silent failures leave jobs in limbo
4. **DB::transaction() in job** — exit/timeout inside transaction won't rollback
5. **Not using unique job IDs** — duplicate dispatches cause double-processing
6. **No retry backoff** — hammer the failing service with immediate retries
7. **Duplicate dispatches causing double-processing** — use `#[DebounceFor]` for bursty workloads where only the last dispatch matters

## Updated from Research (2026-05-08)

- **Queue Routing** — Laravel 13 adds `Queue::route()` for centralized queue/connection routing by job class. Configure once, applies everywhere.
- **Job PHP Attributes** — Laravel 13 introduces `#[Job]`, `#[Job\Backoff()]`, `#[Job\MaxAttempts()]`, `#[Job\Timeout()]`, `#[Job\FailOnTimeout]` as declarative alternatives to job properties.
- **Interruptible Jobs (Laravel 13.7+)** — `ShouldInterrupt` interface lets jobs respond to worker shutdown signals and checkpoint progress for resumable processing.
- **WorkerPausing/WorkerResuming Events (Laravel 13.8+)** — new worker lifecycle events dispatched when queue workers pause or resume. Use to release/reconnect external resources (DB pools, Redis sessions) in coordination with auto-scaling or graceful shutdown.
- **Debounceable Jobs (Laravel 13.6+)** — `#[DebounceFor]` attribute or `->debounce()` at dispatch keeps only the last job in a time window. Eliminates redundant processing from rapid-fire dispatches (e.g., user editing same document 10x in 30s = 1 rebuild).

Source: [Laravel 13 Docs - Queues](https://laravel.com/docs/13.x/queues)
