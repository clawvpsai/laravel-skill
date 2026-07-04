# Logging — Channel Configuration, Structured Logging, Monitoring

## Log Configuration (`config/logging.php`)

```php
'channels' => [
    'stack' => [
        'driver' => 'stack',
        'channels' => ['single', 'daily'],
        'ignore_exceptions' => false,
    ],

    'single' => [
        'driver' => 'single',
        'path' => storage_path('logs/laravel.log'),
        'level' => env('LOG_LEVEL', 'debug'),
    ],

    'daily' => [
        'driver' => 'daily',
        'path' => storage_path('logs/laravel.log'),
        'level' => env('LOG_LEVEL', 'debug'),
        'days' => 14, // retain 14 days
    ],

    'slack' => [
        'driver' => 'slack',
        'url' => env('LOG_SLACK_WEBHOOK_URL'),
        'level' => 'error', // only errors and above
    ],

    'syslog' => [
        'driver' => 'syslog',
        'level' => env('LOG_LEVEL', 'debug'),
    ],

    'errorlog' => [
        'driver' => 'errorlog',
        'level' => env('LOG_LEVEL', 'debug'),
    ],
],
```

## Writing Logs

```php
use Illuminate\Support\Facades\Log;

Log::info('User logged in', ['user_id' => $user->id]);
Log::warning('High memory usage', ['memory_mb' => memory_get_usage() / 1024 / 1024]);
Log::error('Payment failed', ['order_id' => $order->id, 'reason' => $e->getMessage()]);
Log::debug('Query executed', ['sql' => $query, 'time_ms' => $time]);

// Shorthand
Log::info("Processing order {$order->id} for {$user->email}");
```

## Context Across Requests (Shared Logs)

```php
// Set user context that applies to ALL logs until cleared
Log::shareContext(['user_id' => auth()->id(), 'request_id' => $request->id()]);

// In middleware for every request:
class RequestLogMiddleware
{
    public function handle($request, $next)
    {
        $requestId = Str::uuid()->toString();
        Log::shareContext(['request_id' => $requestId]);
        $request->attributes->set('request_id', $requestId);
        return $next($request);
    }
}
```

## Exception Logging with Context

```php
try {
    $order->process();
} catch (PaymentException $e) {
    Log::error('Payment failed', [
        'order_id' => $order->id,
        'user_id' => $order->user_id,
        'amount' => $order->total,
        'exception' => $e,
    ]);
    throw $e;
}
```

## Query Logging (Debug Only)

```php
// In dev environment — logs all queries
DB::listen(fn($query) => Log::debug('SQL', [
    'sql' => $query->sql,
    'bindings' => $query->bindings,
    'time' => $query->time,
]));

// Alternative: Laravel Debugbar (package)
// Alternative: Clockwork (package)
```

## Production Log Monitoring

```php
// Critical errors that need immediate attention
Log::critical('Database connection lost', [
    'host' => config('database.connections.mysql.host'),
    'attempt' => $attempt,
]);
```

**Log levels (lowest to highest):** `debug`, `info`, `notice`, `warning`, `error`, `critical`, `alert`, `emergency`

## Laravel 13 Structured Log Channels

Laravel 13 adds native support for structured log channels:
```php
'papertrail' => [
    'driver' => 'monolog',
    'handler' => Monolog\Handler\SyslogUdpHandler::class,
    'handler_with' => ['host' => env('PAPERTRAIL_URL')],
    'level' => 'debug',
],
```



### JsonFormatter (Laravel 13.6+)

Laravel 13.6 adds a native `JsonFormatter` for Monolog that outputs structured JSON logs — ideal for log aggregation systems (Papertrail, CloudWatch, Datadog):

```php
'channels' => [
    'json' => [
        'driver' => 'monolog',
        'handler' => Monolog\Handler\StreamHandler::class,
        'handler_with' => ['stream' => storage_path('logs/laravel.json.log')],
        'formatter' => Monolog\Formatter\JsonFormatter::class,
        'level' => env('LOG_LEVEL', 'debug'),
    ],
],
```

## `Context` Facade (Laravel 11+ — Recommended Over `shareContext()`)

Laravel ships a first-class `Context` facade for capturing per-request (or per-job, per-command) contextual data that's automatically appended to every log entry and survives the queue boundary:

```php
use Illuminate\Support\Facades\Context;

// Add a single key
Context::add('trace_id', (string) Str::uuid());

// Add multiple at once
Context::add([
    'url'      => $request->url(),
    'hostname' => gethostname(),
    'user_id'  => auth()->id(),
]);

// Add only if not already set
Context::addIf('tenant', $tenantId);

// Read back
$traceId = Context::get('trace_id');
Context::has('trace_id'); // bool
Context::all();           // ['trace_id' => '...', 'url' => '...']
Context::forget('url');   // remove one key
Context::flush();         // clear everything

// Stack semantics — push multiple values under one key
Context::push('job_history', "Job queued: MyFirstJob");
Context::push('job_history', "Job queued: MySecondJob");
Context::get('job_history');
// ['Job queued: MyFirstJob', 'Job queued: MySecondJob']

// Hidden context — included in jobs but NOT written to logs (good for PII or secrets)
Context::addHidden('internal_session_token', $token);
```

**Auto-attached to log entries:**
```
[2026-06-28 18:00:00] production.INFO: Retrieving commits for [laravel/framework] {"repository":"laravel/framework"} {"hostname":"prod-web-1","trace_id":"a158c456-..."}
```
The `{...}` block at the end is the `Context` payload — automatically merged into every log call in scope.

**Survives queued jobs:** when a request dispatches a job, the `Context` state is serialized with the job payload and restored when the worker runs `handle()`. So a `trace_id` set in middleware flows through the HTTP request → queue dispatch → queue handler → outbound HTTP call automatically.

**Hidden context** (`Context::addHidden()`) is also carried to queued jobs but is excluded from log output. Use for tokens, internal IDs, or PII that you need in code but don't want in your log aggregator (Datadog, CloudWatch, etc.).

**Middleware pattern (recommended over `shareContext()`):**
```php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Context;
use Illuminate\Support\Str;
use Symfony\Component\HttpFoundation\Response;

class AssignTraceContext
{
    public function handle(Request $request, Closure $next): Response
    {
        Context::add('trace_id', (string) Str::uuid());
        Context::add('url', $request->url());
        Context::addHidden('api_token', $request->bearerToken());
        return $next($request);
    }
}
```

**`Log::shareContext()` vs `Context::add()`:** `shareContext()` is a legacy alias for `Context::add()`. Prefer the `Context` facade in new code — it's the documented public API, has the full feature set (push, hidden, all, flush), and matches the docs.

## Common Mistakes

1. **`Log::info` in production loops** — spamming logs in tight loops fills disk fast
2. **Not setting log level** — default `debug` is too verbose in production; set `LOG_LEVEL=error` in `.env`
3. **Sensitive data in logs** — never log passwords, tokens, credit card numbers, PII
4. **No log rotation** — use `daily` channel to auto-rotate; `single` eventually fills disk
5. **`var_dump`/`print_r` in code** — use Log facade instead; these don't go to configured channels

## Monitoring Tools (Production)

| Tool | Purpose |
|---|---|
| Sentry | Error tracking, performance monitoring |
| Bugsnag | Error tracking, crash reporting |
| CloudWatch | AWS-native log aggregation |
| Papertrail | Real-time log aggregation |
| Laravel Telescope | Dev/staging debug assistant (not for production) |
| Laravel Horizon | Queue monitoring |

## Updated from Research (2026-05-08)
- Structured logging with `shareContext()` is the recommended approach for request correlation
- Sentry is the most common production error monitoring for Laravel apps
- Laravel 13 Papertrail channel uses Monolog handler natively
- **Channel Name Respected in `on-demand` Log Stacks (Laravel 13.18.1, PR #60635 by @maltf0)** — `Log::build([...])` and `Log::stack([...])` with `driver: 'errorlog'` or `driver: 'monolog'` on an `on-demand` channel previously ignored the configured `channel` name in some callsites (logs landed under the parent's channel instead). Each channel now writes to its own configured name. Affects anyone using `Log::stack(['on-demand-channel' => [...]])` or building dynamic log channels at runtime.
- **JsonFormatter (Laravel 13.6+)** — native `Monolog\Formatter\JsonFormatter` support for structured JSON log output, ideal for log aggregation pipelines

Source: [Laravel Logging](https://laravel.com/docs/13.x/logging)


### `Context` Facade (Laravel 11+)

- `Illuminate\Support\Facades\Context` is the documented public API for per-request/job context that auto-attaches to log entries and survives the queue boundary.
- Methods: `add`, `addIf`, `push`, `get`, `has`, `all`, `forget`, `flush`. `addHidden()` for context that flows to queued jobs but is excluded from log output (PII, tokens).
- `Log::shareContext()` is a thin wrapper over `Context::add()`. Prefer the `Context` facade in new code.
