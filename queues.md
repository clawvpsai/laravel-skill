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
use Illuminate\Foundation\Access\Authorizable;
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