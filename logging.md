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
- **JsonFormatter (Laravel 13.6+)** — native `Monolog\Formatter\JsonFormatter` support for structured JSON log output, ideal for log aggregation pipelines

Source: [Laravel Logging](https://laravel.com/docs/13.x/logging)