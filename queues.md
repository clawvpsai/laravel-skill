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
#[DebounceFor(2, 'minutes')] // explicit units (Laravel 13.14+) — cleaner for larger intervals
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
RebuildDocumentJob::dispatch($docId)->debounce(2, 'minutes'); // with explicit units (Laravel 13.14+)
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

Source: [Laravel 13 PR #60180](https://github.com/laravel/framework/pull/60180)

## Cloud Queue Driver Deepening (Laravel 13.15+)

Several follow-up changes make Laravel Cloud's managed queue driver a first-class connection in 13.15:

- **Managed queues boot before service providers** — jobs are ready before the app finishes booting
- **`ManagedQueueNotFoundException`** thrown when a configured managed queue is missing (was silently absent before)
- **FIFO queue name normalization corrected** — Cloud managed FIFO queues now normalize correctly
- **`Cloud-Request-ID` header emitted in logs** — every log line during cloud queue processing carries the request ID for distributed tracing

```php
// config/queue.php
'connections' => [
    'cloud' => [
        'driver' => 'cloud',
        'connection' => 'managed-queue-uuid', // managed queue ID, not a real driver
    ],
],
```

No app code changes required to upgrade — the existing Cloud driver config works as-is.

Source: [Laravel News — Dedicated Cloud Queue](https://laravel-news.com/laravel-13-15-0) | [PR #60199](https://github.com/laravel/framework/pull/60199) | [PR #60276](https://github.com/laravel/framework/pull/60276)

## `Queue::route()` with Enums (Laravel 13.15+)

`Queue::route()` now accepts enum cases for both queue name and connection, matching how the `#[Queue()]` attribute already supports enums:

```php
use App\Enums\QueueConnection;
use App\Enums\QueueName;
use App\Jobs\ProcessPodcast;
use Illuminate\Support\Facades\Queue;

// String-based (still works)
Queue::route(ProcessPodcast::class, connection: 'redis', queue: 'podcasts');

// Enum-based (Laravel 13.15+) — type-safe routing
Queue::route(
    ProcessPodcast::class,
    connection: QueueConnection::Redis,
    queue: QueueName::Podcasts,
);
```

**Why it matters:** String-based queue names are typo-prone and lose refactor safety. With enums, renaming a queue name becomes a single-point-of-truth rename. Pairs perfectly with the existing `#[Queue(connection: ..., queue: ...)]` attribute support.

## Queue Attributes on Traits (Laravel 13.16+)

PHP attributes like `#[Job]`, `#[Tries]`, `#[Backoff]`, `#[Timeout]`, `#[FailOnTimeout]`, and `#[DebounceFor]` previously only worked when declared directly on the job class. Laravel 13.16 extends attribute discovery to traits:

```php
use Illuminate\Queue\Attributes\DebounceFor;
use Illuminate\Queue\Attributes\Job;
use Illuminate\Queue\Attributes\Tries;

trait DebouncedJob
{
    #[DebounceFor(30, 'seconds')]
}

#[Job]
#[Tries(3)]
class RebuildIndexJob implements ShouldQueue
{
    use DebouncedJob; // #[DebounceFor] now applies from this trait
    // ...
}
```

**Use case:** Build reusable job base classes / traits that bake in retry/backoff/debounce semantics without forcing every concrete job to repeat the attributes.

### PendingDispatch Conditionals (Laravel 13.9+)

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

### Bus::bulk() — Batch Dispatch (Laravel 13.13+)

`Bus::bulk()` dispatches multiple jobs in a single call — useful for batch processing:

```php
use Illuminate\Support\Facades\Bus;

Bus::bulk([
    new ProcessItemJob(1),
    new ProcessItemJob(2),
    new ProcessItemJob(3),
    // ... hundreds more
]);

// With queue/connection override
Bus::bulk([...], queue: 'high-priority', connection: 'redis');
```

**When to use:**
- Bulk imports — process hundreds/thousands of items without a loop of `dispatch()`
- Fan-out operations — same job type across many IDs
- Cleaner than `foreach` + `dispatch()` — one call, one error handling path

**Note:** `Bus::bulk()` is synchronous by default — jobs are dispatched but not necessarily processed yet. Wrap in queue workers as usual.


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

### WorkerIdle Event (Laravel 13.10+)

`WorkerIdle` fires when the queue worker has no jobs to process during a polling cycle. Unlike `WorkerLooping` (which fires on every loop iteration), `WorkerIdle` only fires when the queue is empty — useful for idling down, emitting metrics, or triggering cleanup:

```php
use Illuminate\Queue\Events\WorkerIdle;
use Illuminate\Queue\Events\Looping;
use Illuminate\Support\Facades\Event;
use Illuminate\Queue\WorkerOptions;

Event::listen(WorkerIdle::class, function (WorkerIdle $event, array $payload) {
    [$connection, $queue, $workerOptions] = $payload;

    Log::info("Worker idle — queue empty", [
        'connection' => $connection,
        'queue' => $queue,
    ]);
});
```

**vs WorkerLooping:** `WorkerLooping` fires on every loop (even when jobs exist). `WorkerIdle` only fires when the queue is empty, making it more efficient for "nothing to do" logic.

**Use cases:**
- Scale down worker instances when idle (emit zero-queue-depth metric → trigger scale-in)
- Run cleanup tasks only when the queue is drained
- Emit "queue depth = 0" metrics for dashboards
- Trigger health check to report "idle and healthy"

### `WorkerStopping` Event Now Exposes `jobsProcessed` + `lastJobProcessedAt` (Laravel 13.17+)

Before 13.17, the `WorkerStopping` event had only `$status`, `$workerOptions`, and `$reason` — you couldn't tell how many jobs the worker processed in its lifetime or when the last one finished. As of PR #60592, two new public properties are attached to every stop:

```php
use Illuminate\Queue\Events\WorkerStopping;

Event::listen(WorkerStopping::class, function (WorkerStopping $event) {
    Log::info('queue worker stopping', [
        'reason'             => $event->reason?->value,
        'jobs_processed'     => $event->jobsProcessed,
        'last_job_at'        => $event->lastJobProcessedAt
            ? \Carbon\Carbon::createFromTimestamp($event->lastJobProcessedAt)
            : null,
        'idle_seconds'       => $event->lastJobProcessedAt
            ? now()->getTimestamp() - $event->lastJobProcessedAt
            : null,
    ]);

    // Optional: push to a metric stream for capacity dashboards
    Metrics::gauge('queue.worker.jobs_processed', $event->jobsProcessed);
});
```

**New properties:**
- `$jobsProcessed` (`int|null`) — total jobs the worker handled across the whole process
- `$lastJobProcessedAt` (`int|float|null`) — Unix timestamp of the last job that finished (null if the worker never processed one)
- Existing `$status`, `$workerOptions`, `$reason` are unchanged

**Use cases:**
- Capacity / lifetime dashboards — see how many jobs a worker actually ran before SIGTERM/restart
- Graceful-shutdown debugging — measure the gap between last job and stop signal
- Auto-scaling telemetry — derive worker utilization from `jobsProcessed / uptime`
- Side-effect hooks that need a "last batch checkpoint" trigger

Source: [PR #60592 — expose jobs processed count and last job timestamp on WorkerStopping](https://github.com/laravel/framework/pull/60592)

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

## Bulk SQS via `SendMessageBatch` (Laravel 13.19+, PR #60645)

The SQS queue driver historically sent jobs one-by-one (`SendMessage` per dispatch). 13.19 groups up to 10 dispatches into a single `SendMessageBatch` call, matching SQS's native batch endpoint. **No code change required** — this is a transparent driver-level optimization that activates when you use `Bus::bulk()`, `Queue::bulk()`, or any code that pushes multiple jobs in tight succession.

```php
// Before 13.19: each of these 10 dispatches = 1 SQS API call (SendMessage) = 10 calls total
foreach ($items as $item) {
    ProcessItem::dispatch($item);
}

// 13.19+: the SQS driver batches them into 1 SendMessageBatch call (10 items)
Bus::bulk([
    new ProcessItem($items[0]),
    new ProcessItem($items[1]),
    // ... up to 10 per batch ...
]);
// Subsequent bulk() calls also get batched if the queue worker is configured to drain them
```

**Performance impact:** SQS charges per API call. A 10x reduction on bulk dispatches cuts your SQS cost by ~90% on bulk-heavy apps (data imports, mass notifications, queue migrations). Latency-wise, `SendMessageBatch` is also faster than 10 sequential `SendMessage` calls — SQS processes batches server-side in a single round-trip.

**SQS hard limit:** 10 messages per `SendMessageBatch` call (AWS-imposed). Laravel's driver handles this — if you pass more than 10 jobs in a single `Bus::bulk()`, it splits into multiple `SendMessageBatch` calls of up to 10 each.

**When this helps vs doesn't:**
- Helps: `Bus::bulk([...])`, `Queue::bulk([...])`, any loop dispatching 10+ jobs
- Doesn't help: a single `ProcessItem::dispatch()` — that's still 1 `SendMessage` call
- Doesn't help: `dispatch($job)->onQueue('other-queue')` per item — different queues = different batches

**No configuration needed** — the driver picks up the new behavior automatically. Just upgrade and bulk-dispatch jobs to SQS, and watch your SQS API costs drop.

## Cloud-Agent Queue Pop (Laravel 13.19+, PR #60659 / 12.63.0, PR #60660)

Laravel Cloud's queue driver now reads jobs from the cloud agent's polling endpoint instead of SQS directly when running on Laravel Cloud infrastructure. This is a **deployment-context optimization** — there's nothing to configure in your application code:

```php
// config/queue.php
'connections' => [
    'cloud' => [
        'driver' => 'cloud',  // Laravel Cloud queue driver
        // Before 13.19: driver polled SQS directly
        // 13.19+: driver polls the cloud agent, which polls SQS
    ],
],
```

**Why this matters:**
- **Eliminates duplicate polling.** Each Laravel Cloud app instance was independently polling SQS for jobs. With multiple workers per app (and multiple apps per environment), this multiplied SQS API costs and added "are there jobs?" latency. The cloud agent centralizes the poll.
- **Faster dispatch → execute.** Jobs land in the agent immediately when dispatched. The agent dispatches them to the appropriate worker pool, which can pick them up within milliseconds rather than waiting for the next poll cycle.
- **No app-level changes required.** Pure infrastructure change — upgrade to 13.19+ (or 12.63.0+) on a Laravel Cloud deployment and the driver uses the agent endpoint automatically.

**Detecting the new path in your deployment:**
- Laravel Cloud dashboard will show "queue: agent" instead of "queue: sqs" in the worker metrics tab
- SQS API cost in your AWS bill should drop noticeably for Cloud-hosted apps
- If you self-host (not on Laravel Cloud), the driver falls back to direct SQS polling — unchanged behavior


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

## `ShouldNotRetry` Exception Handler (Laravel 13.17+)

Mark specific exceptions as "never retry this" — when a job throws an exception implementing `ShouldNotRetry`, the queue worker fails the job immediately instead of retrying, regardless of `$tries` / `maxExceptions`. Use for permanent failures (validation errors, missing records, 4xx upstream errors) that retrying cannot fix.

**Two registration paths:**

### 1. Implement on your own exception class

```php
use Illuminate\Contracts\Queue\ShouldNotRetry;

class PaymentGatewayException extends RuntimeException implements ShouldNotRetry
{
    // The interface has no methods — marking it as implemented is enough.
    // The queue worker inspects the exception and skips retries.
}

// Any job that throws PaymentGatewayException will fail immediately,
// consuming 0 additional attempts.
class ChargeCustomerJob implements ShouldQueue
{
    public function handle(): void
    {
        // ...
        throw new PaymentGatewayException("Card declined (no retry possible)");
    }
}
```

### 2. Register a callback in bootstrap for exceptions you don't own

```php
// bootstrap/app.php
->withExceptions(function (Exceptions $exceptions) {
    $exceptions->retry(PaymentGatewayException::class, function () {
        return false; // never retry
    });

    // Multiple exceptions at once
    $exceptions->retry([
        \Illuminate\Validation\ValidationException::class,
        \Illuminate\Database\Eloquent\ModelNotFoundException::class,
        \Symfony\Component\HttpKernel\Exception\NotFoundHttpException::class,
    ], fn () => false);
})
```

**When to use:**
- Validation exceptions — retrying won't make bad input become good
- `ModelNotFoundException` / `NotFoundHttpException` — the record is gone, retry won't find it
- 4xx upstream HTTP errors (vs 5xx) — client error, not transient
- Authentication/authorization exceptions — never going to succeed without code change
- "Permanent" gateway errors (card declined, account closed) — retrying won't help

**Migration from `$tries = 1` + `failOnTimeout`:**
- `ShouldNotRetry` is per-exception, not per-job — more flexible
- Same job class can have some exceptions retry and others fail immediately
- Cleaner than wrapping every handler in try/catch with custom retry logic

Source: [PR #60552](https://github.com/laravel/framework/pull/60552) | [Laravel 13.17 Release Notes](https://github.com/laravel/framework/releases/tag/v13.17.0)

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
8. **Manually calling `$this->release($delay)` in every job** — if your `handle()` is *only* releasing (not actually processing), move the policy into `Illuminate\Queue\Middleware\Release` and attach via `->middleware(new Release($delay))` on the job or its default middleware stack (Laravel 13.18.1+).

## Updated from Research (2026-06-29)

- ** event payload (PR #60592, 13.17+)** — new public  and  properties allow lifetime-job-count and graceful-shutdown-gap dashboards without dirty reflection hacks.


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
- **Dedicated Cloud Queue (Laravel 13.11+)** — `Illuminate\Foundation\Cloud\Queue` decorator wraps any queue driver with cloud-native instrumentation. Tracks processing job, job start timestamps, and dispatches `CloudJobFetching/Processing/Processed/Failed` events. Use via `cloud` queue connection or `Cloud` facade.
- **Cloud-Request-ID (renamed from X-Request-ID, Laravel 13.11+)** — renamed for clarity in cloud deployments. Auto-written to log entries for distributed request tracing.
- **WorkerIdle Event (Laravel 13.10+)** — fires when the queue worker has no jobs to process (only on empty queue, unlike `WorkerLooping` which fires every iteration). Useful for scaling down, cleanup, and idle metrics.
- **assertPushedOnce (Laravel 13.10+)** — `Queue::assertPushedOnce($job)` asserts a job was pushed exactly once. Accepts optional callback for partial matching.
- **`Release` Queue Middleware (Laravel 13.18.1, PR #60630)** — declarative companion to `$this->release($delay)`. Attach via `->middleware(new \Illuminate\Queue\Middleware\Release($delayInSeconds))` so the job releases itself back to the queue after middleware runs, without `handle()` needing to call `release()` manually. Use for "try, and if it would conflict, hand off to another worker" patterns without scattering `release()` calls across job classes. Add to `withMiddleware()` in `bootstrap/app.php` or apply per-job via `->middleware(new Release(60))` on the dispatch chain.
- **Inspect Delayed Jobs on `Queue::fake()` (Laravel 13.18.1, PR #60636)** — `Queue::fake()` now tracks the `availableAt` timestamp on pushed jobs. Tests can inspect delayed releases via `Queue::assertReleased()` / `Queue::assertDelayed()` instead of losing the delay metadata when reading the fake.

Source: [Laravel 13 Docs - Queues](https://laravel.com/docs/13.x/queues)

### Laravel Reverb Database Driver — WebSocket Without Redis (Laravel 13)

Laravel Reverb now ships with a **native database driver**, eliminating the Redis dependency for real-time WebSocket features. Previously, Reverb required Redis for its internal pub/sub layer. Now you can run full-featured WebSockets with just MySQL/PostgreSQL.

**Why it matters:**
- Managed VPS / shared hosting deployments no longer need Redis for real-time features
- Simpler infrastructure for smaller apps — reduces operational cost and complexity
- Pusher is no longer the only "easy" option for real-time without self-hosting Redis

**Configuration (`config/broadcasting.php` or `.env`):**
```php
// .env
BROADCAST_CONNECTION=reverb
REVERB_DRIVER=database

# Optional: tune for database driver
REVERB_DB_POLL_INTERVAL=5000  # polling interval in ms
```

**Run migrations for Reverb's database tables:**
```bash
php artisan reverb:table
php artisan migrate
```

**Start Reverb server:**
```bash
php artisan reverb:start
```

**Compare driver options:**

| Driver | Best For | Infrastructure |
|---|---|---|
| **Redis** | High-traffic, multi-server | Redis server required |
| **Database** | Medium traffic, single/simple setup | MySQL/PostgreSQL only |
| **Log** | Debugging only | None extra |

**Performance note:** Database driver uses polling, which is less efficient than Redis pub/sub at scale. For <1,000 concurrent WebSocket connections, database driver is typically fine. Beyond that, add Redis.

**Reverb in deployment (Supervisor):**
```ini
[program:reverb]
command=php /path-to-project/artisan reverb:start
numprocs=1
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/var/log/reverb.log
```

Source: [Laravel 13 Docs - Reverb](https://laravel.com/docs/13.x/reverb)

---

## Dedicated Cloud Queue (Laravel 13.11+)

`Illuminate\Foundation\Cloud\Queue` is a **decorator wrapper** around any existing queue driver (Redis, SQS, database, etc.) that adds cloud-native instrumentation:

- Tracks the currently processing job and when it started
- Dispatches `CloudJobFetching`, `CloudJobProcessing`, `CloudJobProcessed`, `CloudJobFailed` events
- Logs `Cloud-Request-ID` alongside job processing for distributed tracing
- Provides structured metrics via `Queue::connection('cloud')->metrics()`

**How it works:**
```php
// config/queue.php
'connections' => [
    'cloud' => [
        'driver' => 'cloud',
        'connection' => 'redis',   // underlying driver to wrap
        'queue' => env('AWS_SQS_QUEUE_URL'),
        'region' => env('AWS_DEFAULT_REGION'),
    ],
],
```

```php
// .env
QUEUE_CONNECTION=cloud
```

**Or wrap at dispatch time using the Cloud facade:**
```php
use Illuminate\Support\Facades\Cloud;

Cloud::connection('sqs')->queue('emails')->push(new SendEmailJob($user));
```

**Dedicated Cloud Queue events:**
```php
use Illuminate\Foundation\Cloud\Events\CloudJobFetching;
use Illuminate\Foundation\Cloud\Events\CloudJobProcessing;
use Illuminate\Foundation\Cloud\Events\CloudJobProcessed;
use Illuminate\Foundation\Cloud\Events\CloudJobFailed;

Event::listen(CloudJobProcessing::class, function (CloudJobProcessing $event) {
    Log::info("Cloud job processing", [
        'job' => $event->job->connectionName,
        'started_at' => $event->startedAt->toIso8601String(),
        'cloud_request_id' => $event->cloudRequestId,
    ]);
});
```

**CloudFailedJobProvider:**
Cloud queue includes a dedicated `FailedJobProvider` that stores failure metadata (exception class, message, timestamps, cloud request ID) for cross-service correlation:
```php
// config/queue.php
'failed' => [
    'driver' => 'cloud',
    'provider' => 'database',  // underlying failed job storage
],
```

**Use cases:**
- Multi-service job tracing with `Cloud-Request-ID` correlation
- Per-job latency tracking (job started → job finished → latency metric)
- Cloud-native failed job storage with rich metadata
- Foundation for managed queue platforms (Laravel Cloud, Vapor, etc.)

Source: [Laravel 13 PR #60180](https://github.com/laravel/framework/pull/60180)
