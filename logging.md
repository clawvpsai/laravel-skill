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

**CRITICAL gotcha — enable stack traces or you lose them in JSON:** Monolog's `JsonFormatter` strips exception stack traces by default to keep log lines compact. Structured log aggregators (Datadog, Loki, Elastic, CloudWatch Logs Insights) need the stack trace as a searchable string field. Pass `formatter_with` to opt in:

```php
'channels' => [
    'json' => [
        'driver' => 'monolog',
        'handler' => Monolog\Handler\StreamHandler::class,
        'handler_with' => ['stream' => storage_path('logs/laravel.json.log')],
        'formatter' => Monolog\Formatter\JsonFormatter::class,
        // Required for stack traces in JSON logs — without this, errors are logged as
        // {"context": {"exception": "[object] (Exception...)"}} with the stack gone
        'formatter_with' => [
            'includeStacktraces' => true,
        ],
        'level' => env('LOG_LEVEL', 'debug'),
    ],
],
```

**Quick check — search for the missing stack trace in your aggregator:** if a `Log::error($e)` in code produces a JSON line with `"exception": "[object] (RuntimeException...)"` and no `"trace": "..."` field, `includeStacktraces` is off. Fly.io [documents this gotcha](https://fly.io/docs/laravel/the-basics/logging-stack-traces/) — Laravel Cloud ships `includeStacktraces => true` by default; on-prem FrankenPHP/Swoole does not. Affects **every** JSON-formatted Monolog channel.

**Alternative for production:** pair with Monolog's `JsonFormatter::BATCH_MODE_JSON` and `appendNewline: true` for newline-delimited JSON (NDJSON) — Datadog, Loki, and Elastic ingest this format directly without parsing:
```php
'formatter_with' => [
    'batchMode' => Monolog\Formatter\JsonFormatter::BATCH_MODE_JSON,
    'appendNewline' => true,
    'includeStacktraces' => true,
],
```

## Tail & Filter Logs with Pail (Laravel 11+, official in Laravel 13)

Laravel Pail is the first-party log-tail CLI — it works with **any** log driver (file, Sentry, Flare, Nightwatch) and lets you filter by exception type, stack trace content, message, and level in real time. Far more useful than `tail -f` because it knows the structured fields.

**Install:**
```bash
# Laravel 13 — pail ships with the framework; nothing to install
# Laravel 11/12 — install as a dev dependency:
composer require --dev laravel/pail
# Requires the PCNTL PHP extension (install with: apt install php8.3-pcntl)
```

**Basic usage:**
```bash
php artisan pail              # tail with default verbosity (truncates long lines)
php artisan pail -v           # show full lines (no truncation)
php artisan pail -vv          # MAX verbosity — shows exception STACK TRACES inline
php artisan pail --level=error          # only errors and above
php artisan pail --message="User created"     # exact message match
php artisan pail --filter="QueryException"    # searches type, file, message, AND stack trace content
php artisan pail --filter="Stripe" --level=error --no-interaction
```

**Why `--filter` is the killer feature:** it matches against the exception **stack trace** too — `php artisan pail --filter="OrderRepository"` will surface every log line whose stack trace ran through `OrderRepository.php`, even if the log message is generic. `grep` + JSON logs loses this; Pail keeps it.

**Pail works with structured log channels:** unlike `tail -f` (which assumes line-based text), Pail understands JSON output, Sentry's exceptions, Flare's groupings, and Nightwatch's stream. If you're tailing a JSON log file, Pail parses each line and lets `--filter` match against nested fields.

**Use Pail over `tail -f` when:**
- The log file is JSON-formatted (Datadog-style NDJSON, see JsonFormatter section above)
- You need to filter by exception class or file path
- You're debugging a specific user/action: `pail --filter="$traceId"`

**Use `tail -f` only when:**
- You're reading a human-formatted `single` or `daily` channel output and want literal byte-level tails
- You're piping the output to grep/awk downstream

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

## Deprecation Warnings Channel (`LOG_DEPRECATIONS_CHANNEL`)

PHP, Laravel, and Composer all emit deprecation warnings when you use features that will be removed in future versions. By default these go to `null` (silently swallowed). Configure a dedicated channel to surface them — invaluable when preparing for a major Laravel upgrade.

```php
// config/logging.php
return [
    'deprecations' => [
        'channel' => env('LOG_DEPRECATIONS_CHANNEL', 'null'),
        'trace' => env('LOG_DEPRECATIONS_TRACE', false),
    ],

    'channels' => [
        // ... your other channels

        'deprecations' => [
            'driver' => 'single',
            'path' => storage_path('logs/php-deprecation-warnings.log'),
            'level' => 'debug',
        ],
    ],
];
```

```bash
# .env
LOG_DEPRECATIONS_CHANNEL=deprecations   # point at the channel name above
LOG_DEPRECATIONS_TRACE=true            # include the stack trace of where the deprecation was triggered
```

**FOOTGUN — `'null'` is a string, not PHP `null`:** the line `'channel' => env('LOG_DEPRECATIONS_CHANNEL', 'null')` defaults to the **literal string** `'null'`, not PHP `null`. Laravel resolves that as a channel name and tries to find a channel called `null` — if you haven't defined one, you get `Log [deprecations] is not defined` errors (see [php/frankenphp #1484](https://github.com/php/frankenphp/issues/1484)). The literal `'null'` works only because Laravel ships a built-in null channel by that name. **When in doubt, override the env var to your own channel name.**

**Why you want this:**
- Catch PHP 8.x deprecations (implicit nullable params, dynamic props, etc.) before PHP 9 ships
- Find every place in your code that uses a soon-deprecated Laravel API (e.g. `Str::random(32)` vs `Str::random(length: 32)`)
- Audit third-party packages — a noisy `deprecations.log` is the first signal that an upgrade will break

**Don't leave `trace=false` forever:** the warning itself tells you what's deprecated; the trace tells you **where** in your codebase. Without `trace=true`, you'll see `Implicitly marking parameter $x as nullable is deprecated` 50 times in `laravel/framework` without knowing which of your controllers triggered it. Set `LOG_DEPRECATIONS_TRACE=true` in non-production first to confirm; turn it on in production only if you ship the deprecation log to a dedicated aggregator (don't pollute the main error pipeline).

## Common Mistakes

1. **`Log::info` in production loops** — spamming logs in tight loops fills disk fast
2. **Not setting log level** — default `debug` is too verbose in production; set `LOG_LEVEL=error` in `.env`
3. **Sensitive data in logs** — never log passwords, tokens, credit card numbers, PII
4. **No log rotation** — use `daily` channel to auto-rotate; `single` eventually fills disk
5. **`var_dump`/`print_r` in code** — use Log facade instead; these don't go to configured channels
6. **`JsonFormatter` without `includeStacktraces`** — production logs lose the stack trace; aggregators can't link errors to source code (see above)
7. **`LOG_DEPRECATIONS_CHANNEL` left at default** — deprecation warnings silently disappear, you find out PHP 9 broke your app when it ships
8. **Setting `LOG_DEPRECATIONS_CHANNEL=null` thinking it disables the channel** — it's the string `'null'`, Laravel resolves to a channel called `null`; set `LOG_DEPRECATIONS_CHANNEL=` (empty) or omit it entirely if you genuinely want to silence
9. **`tail -f` on JSON logs** — you can't filter by exception class or trace; use `php artisan pail --filter=...` instead
10. **Forgetting to add `request_terminate_timeout` at FPM layer** — see `deployment.md` § PHP-FPM Pool Hardening; long stack-trace logging under low `php_admin_value[memory_limit]` can OOM a worker mid-write

## Monitoring Tools (Production)

| Tool | Purpose |
|---|---|
| Sentry | Error tracking, performance monitoring |
| Bugsnag | Error tracking, crash reporting |
| CloudWatch | AWS-native log aggregation |
| Papertrail | Real-time log aggregation |
| Laravel Telescope | Dev/staging debug assistant (not for production) |
| Laravel Horizon | Queue monitoring |
| **Laravel Nightwatch** | First-party observability (queries, jobs, requests, exceptions, slow logs) — auto-instrumented SaaS; pairs directly with `php artisan pail` for live tail |
| Datadog / New Relic / Honeycomb | APM + log aggregation; consume `JsonFormatter` NDJSON output (see JsonFormatter section) |
| Flare | First-party error tracking from the makers of Spatie (free tier); integrates with Pail |

## Updated from Research (2026-07-18 — cycle 45)
- Structured logging with `shareContext()` is the recommended approach for request correlation
- Sentry is the most common production error monitoring for Laravel apps
- Laravel 13 Papertrail channel uses Monolog handler natively
- **Channel Name Respected in `on-demand` Log Stacks (Laravel 13.18.1, PR #60635 by @maltf0)** — `Log::build([...])` and `Log::stack([...])` with `driver: 'errorlog'` or `driver: 'monolog'` on an `on-demand` channel previously ignored the configured `channel` name in some callsites (logs landed under the parent's channel instead). Each channel now writes to its own configured name. Affects anyone using `Log::stack(['on-demand-channel' => [...]])` or building dynamic log channels at runtime.
- **JsonFormatter (Laravel 13.6+)** — native `Monolog\Formatter\JsonFormatter` support for structured JSON log output, ideal for log aggregation pipelines
- **JsonFormatter `formatter_with => ['includeStacktraces' => true]` (CRITICAL production setting)** — without this, JSON stack traces are stripped to just the exception message; aggregators (Datadog/Loki/Elastic/CloudWatch Logs Insights) lose the ability to link errors to source code. Laravel Cloud ships it on by default; on-prem FrankenPHP/Swoole/RoadRunner setups do not. Fly.io [documents this gotcha](https://fly.io/docs/laravel/the-basics/logging-stack-traces/) for their Laravel deploy image.
- **`php artisan pail` for live log tailing with filters** — ships with Laravel 13, installable via `composer require --dev laravel/pail` on 11/12 (requires PCNTL extension). `--filter` matches against exception type, file, message, AND stack trace content (the killer feature `tail -f + grep` can't replicate on JSON logs); `--message` exact-match; `--level=error` minimum threshold; `-vv` displays full exception stack traces inline. Works against file, Sentry, Flare, and Nightwatch backends.
- **`LOG_DEPRECATIONS_CHANNEL` + `LOG_DEPRECATIONS_TRACE`** — official first-party config for capturing PHP/Laravel/Composer deprecation warnings to a dedicated file (defaults to the string `'null'`, a real footgun — see Common Mistakes #7-8). Critical pre-Laravel-upgrade hygiene. Pair with `trace=true` to identify which line of your codebase triggers each warning before PHP 9 ships.
- **Laravel Nightwatch** — first-party Laravel observability SaaS from the framework team; auto-instruments queries, jobs, requests, exceptions, and slow logs. Pairs directly with `php artisan pail` for live tailing.

Source: [Laravel Logging](https://laravel.com/docs/13.x/logging) | [Pail docs](https://laravel.com/docs/13.x/logging#tailing-log-messages-using-pail) | [Laravel Context docs](https://laravel.com/docs/13.x/context) | [Fly.io stack trace logging guide](https://fly.io/docs/laravel/the-basics/logging-stack-traces/) | [LOG_DEPRECATIONS_CHANNEL footgun — php/frankenphp #1484](https://github.com/php/frankenphp/issues/1484) | [PR #60635 — on-demand log channel name fix](https://github.com/laravel/framework/pull/60635)


### `Context` Facade (Laravel 11+)

- `Illuminate\Support\Facades\Context` is the documented public API for per-request/job context that auto-attaches to log entries and survives the queue boundary.
- Methods: `add`, `addIf`, `push`, `get`, `has`, `all`, `forget`, `flush`. `addHidden()` for context that flows to queued jobs but is excluded from log output (PII, tokens).
- `Log::shareContext()` is a thin wrapper over `Context::add()`. Prefer the `Context` facade in new code.

### `php artisan pail` — Live Log Tail with Filters (Laravel 11+)

- Ships with Laravel 13; install on 11/12 with `composer require --dev laravel/pail` (requires the PCNTL extension).
- Works against **any** log driver — file, Sentry, Flare, Nightwatch — not just filesystem channels. `--filter` matches against exception type, file, message, and stack trace content (the `grep`-equivalent that survives JSON logs).
- `-vv` prints full exception stack traces inline (`-v` for full-line no-truncation, default truncates long lines with `…`).
- Compared to `tail -f`: Pail knows structure (JSON parsing, exception class), supports filters, and pairs naturally with the structured log channels described above. Use `tail -f` only for literal byte-level tails of human-formatted files.

### `JsonFormatter` + `includeStacktraces` (Production JSON Stack Traces)

- `Monolog\Formatter\JsonFormatter` strips stack traces by default. For production JSON log aggregation (Datadog, Loki, Elastic, CloudWatch Logs Insights) you MUST pass `formatter_with => ['includeStacktraces' => true]` or your aggregator loses the line that links an error to source code.
- Pair with `batchMode => BATCH_MODE_JSON` + `appendNewline => true` for newline-delimited JSON (NDJSON) that aggregators ingest natively.
- Server-less and edge platforms (Laravel Cloud) ship this on by default; self-hosted FrankenPHP/Swoole/RoadRunner do not — easy to miss until you grep your first error.

### `LOG_DEPRECATIONS_CHANNEL` Deprecation Channel

- Set `LOG_DEPRECATIONS_CHANNEL=` to your own `'deprecations'` channel name in `config/logging.php` to capture PHP, Laravel, and Composer deprecation warnings to a dedicated `storage/logs/php-deprecation-warnings.log`.
- `LOG_DEPRECATIONS_TRACE=true` adds the full stack trace so you can locate *which line of your code* triggered the warning, not just what was deprecated. Indispensable pre-upgrade audit (PHP 9, Laravel 14).
- **Footgun:** the default is the literal **string** `'null'`, not PHP `null` — Laravel resolves it as a channel name and tries to find a `null` channel (which exists as a built-in NullChannel). Setting `LOG_DEPRECATIONS_CHANNEL=null` in `.env` overrides to that channel name, NOT a disabled state. See Common Mistakes #7-8.
