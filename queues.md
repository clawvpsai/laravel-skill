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



### PreparesForDispatch Interface (Laravel 13.9+)

Jobs can implement `PreparesForDispatch` to run synchronous validation/enrichment logic at dispatch time — before the job hits the queue. Useful for early rejection, enrichment, or checks that should run in the request context rather than the async worker:

```php
use Illuminate\Contracts\Queue\PreparesForDispatch;
use Illuminate\Queue\Attributes\Job;

#[Job]
#[Job\MaxAttempts(3)]
class SendWelcomeEmailJob implements ShouldQueue, PreparesForDispatch
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(public int $userId) {}

    public function prepareForDispatch(): void
    {
        // Runs synchronously when dispatch() is called — before queuing
        // Throw exception here to reject the dispatch entirely
        $user = User::find($this->userId);
        if (!$user || !$user->is_active) {
            throw new \RuntimeException("Cannot dispatch: user {$this->userId} is invalid or inactive");
        }
        // Enrich job data
        $this->userId = $user->id; // normalize ID
    }

    public function handle(): void
    {
        // Runs async in worker — data already validated and enriched
        Mail::to($this->user)->send(new WelcomeMail($this->user));
    }
}
```

**When to use:**
- Validate prerequisites before the job is queued (fail fast, don't queue broken jobs)
- Enrich or normalize job data in the request context
- Early-exit if a business rule disqualifies the job (e.g., user unsubscribed, feature flag off)
- Throttle at dispatch site without custom middleware

**Key difference from `__construct`:** `prepareForDispatch()` runs before serialization and queueing. `__construct` runs during dispatch but may serialize arguments. Use `prepareForDispatch` for anything that must be validated against live data right before queuing.

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

### Worker Timeout Exit Code Override (Laravel 13.9+)

By default, a timed-out worker exits with code `0`. Laravel 13.9 lets you override this via the `--exit-code` flag, so your process supervisor can distinguish timeout exits from normal exits:

```bash
# Exit with code 2 on timeout (distinct from normal exit code 0)
php artisan queue:work redis --timeout=60 --exit-code=2
```

```ini
# supervisor: treat exit code 2 as FAIL, not success
command=php /path/to/artisan queue:work redis --timeout=60 --exit-code=2
exitcodes=2
```

**Use cases:**
- Distinguish timeout exits from success in supervisor `exitcodes` config
- Trigger auto-restart policy only on genuine failures, not expected timeouts
- Health check scripts that grep exit codes for alerting

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

## SQS Large Payload Disk Storage (Laravel 13.9+)

SQS has a 256KB message size limit. For jobs with large payloads (serialized models, bulk data), Laravel 13.9 can offload the body to a local disk (or S3) and send only a reference through SQS:

```php
// config/queue.php
'connections' => [
    'sqs' => [
        'driver' => 'sqs',
        'key' => env('AWS_ACCESS_KEY_ID'),
        'secret' => env('AWS_SECRET_ACCESS_KEY'),
        'region' => env('AWS_DEFAULT_REGION'),
        'queue' => env('SQS_QUEUE'),
        'payload_disk' => 'local', // 's3' also supported
        // or 'payload_disk' => 's3' with bucket config
    ],
],
```

**With S3 disk:**
```php
'connections' => [
    'sqs' => [
        'driver' => 'sqs',
        // ... standard SQS config ...
        'payload_disk' => 's3',
        'payload_disk_config' => [
            'disk' => 's3',
            'path_prefix' => 'queue-payloads/',
        ],
    ],
],
```

**How it works:**
- Payload > 256KB → stored on disk/S3, reference key sent via SQS
- Worker retrieves reference → fetches actual payload from disk/S3 → processes job
- Automatic cleanup after successful processing

**When to use:**
- Jobs that need to pass large data (bulk imports, report generation payloads)
- Avoids SQS extended client library manual setup
- Local disk is fast but ephemeral (use S3 for multi-worker production)

## Cloud Queue Metrics (Laravel 13.9+)

Laravel 13.9 introduces first-party cloud queue metric support for SQS and other cloud providers. Metrics (jobs received, processed, failed, latency) are exposed via the Queue facade for observability:

```php
use Illuminate\Support\Facades\Queue;
use Illuminate\Queue\CloudQueueMetrics;

// Access metrics for a connection
$metrics = Queue::connection('sqs')->metrics();

// Returns metrics object with:
// $metrics->jobsReceived()   — total jobs pushed to queue
// $metrics->jobsProcessed()  — total jobs successfully handled
// $metrics->jobsFailed()     — total failed jobs
// $metrics->averageLatency() — avg time from receive to complete (ms)
```

**Use in monitoring dashboards / health checks:**
```php
Route::get('/queue-health', function () {
    $metrics = Queue::connection('sqs')->metrics();

    return response()->json([
        'queue' => 'sqs',
        'jobs_received' => $metrics->jobsReceived(),
        'jobs_processed' => $metrics->jobsProcessed(),
        'jobs_failed' => $metrics->jobsFailed(),
        'avg_latency_ms' => $metrics->averageLatency(),
        'healthy' => $metrics->jobsFailed() < 10, // custom threshold
    ]);
});
```

**Use cases:**
- Health check endpoints that include queue depth/failure rate
- Alerting when job failure rate spikes above threshold
- Tracking queue backpressure and latency trends
- Capacity planning (are workers keeping up with incoming volume?)

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

## Updated from Research (2026-05-15)

- **SQS Large Payload Disk Storage (Laravel 13.9+)** — payloads exceeding SQS 256KB limit are offloaded to local disk or S3 automatically. Configure via `payload_disk` and `payload_disk_config` in the SQS queue connection. Eliminates manual setup of SQS Extended Client.
- **Cloud Queue Metrics (Laravel 13.9+)** — first-party `Queue::connection()->metrics()` API exposes `jobsReceived()`, `jobsProcessed()`, `jobsFailed()`, and `averageLatency()` for monitoring dashboards and health checks.
- **Worker Timeout Exit Code Override (Laravel 13.9+)** — `queue:work --exit-code=N` lets supervisors distinguish timeout exits from normal exits via distinct exit codes, enabling smarter restart/alarm policies.
- **PreparesForDispatch Interface (Laravel 13.9+)** — `PreparesForDispatch` interface on jobs runs synchronous validation/enrichment logic at dispatch time (before queuing). Use for early rejection, data normalization, or prerequisite checks that should fail fast in the request context rather than async in the worker.
- **PendingDispatch Conditionals (Laravel 13.9+)** — `PendingDispatch` supports `->when()` and `->unless()` for declarative conditional dispatch. Route premium users to dedicated queues without custom logic.
- **Queue Routing (Laravel 13)** — Laravel 13 adds `Queue::route()` for centralized queue/connection routing by job class. Configure once, applies everywhere.
- **Job PHP Attributes (Laravel 13)** — `#[Job]`, `#[Job\Backoff()]`, `#[Job\MaxAttempts()]`, `#[Job\Timeout()]`, `#[Job\FailOnTimeout]` as declarative alternatives to job properties.
- **Interruptible Jobs (Laravel 13.7+)** — `ShouldInterrupt` interface lets jobs respond to worker shutdown signals and checkpoint progress for resumable processing.
- **WorkerPausing/WorkerResuming Events (Laravel 13.8+)** — new worker lifecycle events for coordinating external resources during pause/resume cycles.
- **Debounceable Jobs (Laravel 13.6+)** — `#[DebounceFor]` attribute or `->debounce()` at dispatch keeps only the last job in a time window.

Source: [Laravel 13 Docs - Queues](https://laravel.com/docs/13.x/queues)
