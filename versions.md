# Laravel Version-Specific Requirements

## Active Versions

- **Laravel 11** — Still receives security fixes
- **Laravel 12** — Active development
- **Laravel 13** — Current latest (v13.8.0 as of May 2026)

## Version Selector Prompt

When working with Laravel, start by determining the version. Ask the user or check `composer.json`:
```bash
grep '"laravel/framework"' composer.json | grep -oP '\d+\.\d+'
```

Then load the relevant sections below.

---

## Laravel 13 (Latest — March 2026, v13.8.0)

### New in Laravel 13

- **PHP 8.3 minimum required** (8.2, 8.3, 8.4, 8.5 supported)
- **Laravel AI / Boost MCP** — first-party MCP server for AI assistants
  - `/upgrade-laravel-v13` slash command for automated upgrades
  - Interactive HTML responses in Claude/VS Code Copilot
- **Improved Eloquent** — better attribute casting, faster queries
- **Faster routing** — optimized route resolution
- **Native TypeScript support** — improved scaffolding
- **Laravel Reverb** — first-party WebSocket server (production-ready)
- **New PHP Attributes for Controllers:** `#[Middleware]` and `#[Authorize]`
- **New PHP Attributes for Testing:** `#[Group]`, `#[TestProperty]`, `#[UnitTest]`
- **New PHP Attributes for Queues:** `#[Job]`, `#[Job\Backoff()]`, `#[Job\MaxAttempts()]`, `#[Job\Timeout()]`, `#[Job\FailOnTimeout]`
- **Queue Routing** — `Queue::route()` for centralized queue/connection routing by job class

### New in Laravel 13.8 (May 2026)

- **DatabaseLock excludes expired locks** — `DatabaseLock::isLock` now properly excludes expired locks when checking ownership, fixing race conditions in lock cleanup
- **Worker Pausing/Resuming events** — new `WorkerPausing` and `WorkerResuming` events dispatched by queue workers
- **assertSessionMissingInput** — new TestResponse assertion to verify old input is NOT present in session after form submission
- **assertPushedOn enum support for QueueFake** — `QueueFake::assertPushedOn()` now accepts an enum as the queue argument
- **Custom ON DELETE/UPDATE for foreign keys** — `foreignId()->constrained()->onDelete()->onUpdate()` now supports fully custom constraint actions
- **LocalScope private recursion fix** — `LocalScope` now properly handles private methods without infinite recursion
- **schedule:list environment filter** — `php artisan schedule:list --environment=production` filters output by environment
- **Collection min/max generic result type** — `Collection::min()`/`max()` now return `T|null` instead of `mixed`
- **All queue inspection methods** — new `allPushed()`, `allNotPushed()`, `allPushedOn()` methods on QueueFake for batch assertions

### New in Laravel 13.7 (April 2026)

- **Interruptible Jobs** — `ShouldInterrupt` interface for jobs that respond to worker shutdown signals and checkpoint progress for resumable processing
- **@fonts Blade Directive** — generates `<link rel="preload">` tags for Vite-managed fonts, improving Core Web Vitals
- **Bulk JSON Path Assertions** — `assertJsonPaths()` on TestResponse for asserting multiple paths at once
- **SortDirection Enum** — typed enum for sort direction in query builders



### New in Laravel 13.6 (April 2026)

- **Debounceable Queued Jobs** — `#[DebounceFor]` attribute keeps only the last dispatch within a time window. Eliminates redundant processing from bursty workloads (e.g., user edits same doc 10x in 30s → 1 rebuild instead of 10).
- **JSON support for health route** — `/up` health endpoint now supports returning JSON response data for richer health status
- **JsonFormatter** — native `Monolog\Formatter\JsonFormatter` support for structured JSON log output
- **Cloudflare Email Service support** — new mail driver integration for Cloudflare's email routing

### New in Laravel 13.5 (Late April 2026)

- Queue infrastructure maturity improvements across v13.3 through v13.6 releases

### Breaking Changes from 12

- PHP 8.2 minimum (8.3 recommended)
- `assertStatus()` fully removed (use `assertSuccessful()` or `assertStatus(200)`)
- Some deprecated helpers removed
- `config/app.php` structure may need verification

### Migration to Laravel 13

```bash
composer require laravel/framework:^13.0 --with-all-dependencies
php artisan --version  # verify
```

### Laravel 13 AI Features

```bash
# Install Boost MCP server
php artisan boost:install

# Use AI-assisted upgrade
/upgrade-laravel-v13
```

---

## Laravel 12 (February 2025)

### New in Laravel 12

- **Application Starter Kits** — React, Svelte, Vue, and Livewire with Shadcn components
- **PHP 8.2 minimum required** — supports 8.2, 8.3, 8.4, 8.5
- **Bootstrap 5 by default** — no more Bootstrap 4
- **Vite as default bundler** — Laravel Mix officially deprecated
- **Health endpoint at `/up`** — built into framework, no route needed
- **`php artisan serve` accepts `--host` and `--port`** natively
- **`Queue::after()` and `Queue::failure()` methods** — cleaner queue events
- **Per-second rate limiting** — `RateLimiter::for('api', ...)` supports `perSecond()`
- **`Str::stripTags()`** — better than `strip_tags()`
- **Laravel Reverb** — first-party WebSocket server

### Breaking Changes from 11

- `bootstrap/app.php` no longer registers all service providers by default
- `config/app.php` no longer has `aliases` array
- Route middleware registered in `bootstrap/app.php` via `$middleware->`
- `App\Http\Kernel` and `App\Console\Kernel` fully removed

---

## Laravel 11 (2024)

### New in Laravel 11

- **Application skeleton cleaned** — far less boilerplate
- **`bootstrap/app.php`** replaces `Kernel.php` for middleware registration
- **`config/app.php`** much shorter — providers and aliases removed
- **Per-second rate limiting** via `RateLimiter::for()` with `->perSecond()`
- **Health check endpoint** — `Route::get('/up', fn() => 'ok')`
- **`php artisan about`** shows comprehensive environment summary
- **Password rehashing** — automatic on login
- **`assertSuccessful()`** on responses

### Breaking Changes from 10

- `App\Http\Kernel` removed — use `bootstrap/app.php`
- `App\Console\Kernel` removed — use `routes/console.php`

---

## Laravel 10 (2023 — LTS, Security until late 2026)

### New in Laravel 10

- **PHP 8.1 minimum required**
- **`Illuminate/Validation/Rules/ProhibitedBy`**
- **Native rate limiting** — `RateLimiter` in `RouteServiceProvider`
- **`RateLimiter::for()`** accepts callable returning `Limit` objects
- **Scheduled tasks on one server** — `onOneServer()` method
- **`Model::preventsAccessingMissingAttributes()`** — lazy-loading guard

---

## Version-Agnostic Best Practices (Always Apply)

### Always Use

- PHP 8.3+ (Laravel 13 requires 8.3+, optimized for 8.4)
- Composer 2.x
- Vite (not Mix — Mix deprecated in 11+)
- SQLite for local dev
- `php artisan key:generate` on fresh install
- `assertSuccessful()` not `assertStatus(200)` in tests

### Deprecation Timeline

| Feature | Deprecated In | Removed In |
|---|---|---|
| Laravel Mix | 11 (use Vite) | 12 |
| `assertStatus()` | 10 | 13 |
| `php artisan make:model -mcr` shorthand | Still works | — |
| `App\Http\Kernel` removed | 11 | 12 |
| `App\Console\Kernel` removed | 11 | 12 |
| `config/app.php` aliases array | 11 | 12 |
| PHP 8.0/8.1/8.2 support | 13 | — |
| Bootstrap 4 default | 12 | — |

### Version Detection

```bash
# Check Laravel version
php artisan --version
# Laravel 13.8.0

# Check PHP version
php -v
# PHP 8.4.x

# Check composer
composer --version
# Composer 2.x
```

---

## Agent Checklist Per Task

Before working on any Laravel task:
- [ ] Identify Laravel version (`php artisan --version`)
- [ ] Check PHP version compatibility (8.3+ for Laravel 13)
- [ ] Load version-specific rules above
- [ ] Check if feature is version-specific
- [ ] Apply version-specific patterns, not generic ones

## Updated from Research (2026-05-08)

### Laravel 13.8 (May 2026) — Latest Patch

- **DatabaseLock expired lock fix** — `DatabaseLock::isLock` now excludes expired locks
- **Worker events** — new `WorkerPausing`/`WorkerResuming` events
- **assertSessionMissingInput** — test session old input is absent after form submit
- **QueueFake assertPushedOn enum** — accepts enum as queue name
- **Foreign key onDelete/onUpdate custom actions** — `->onDelete()->onUpdate()` now fully customizable
- **schedule:list --environment filter** — filter schedule list by environment
- **Collection min/max generic types** — proper `T|null` return types
- **All queue inspection methods** — `allPushed()`, `allNotPushed()`, `allPushedOn()` on QueueFake

### Laravel 13.8 (May 2026) — Latest Patch (see above)

### Laravel 13.7 (April 2026)

- **Interruptible Jobs** — `ShouldInterrupt` interface for graceful worker shutdown + checkpointing
- **@fonts Blade Directive** — preload Vite-managed fonts for better Core Web Vitals
- **Bulk JSON Path Assertions** — `assertJsonPaths()` for multi-path JSON testing
- **SortDirection Enum** — typed enum for query builder sorting

### Laravel 13.6 (April 2026)

- **Debounceable Jobs** — `#[DebounceFor]` attribute or `->debounce()` keeps only the last job dispatch in a time window. Clean answer for bursty workloads — user editing same document 10 times in 30 seconds triggers 1 rebuild, not 10.
- **JsonFormatter** — native `Monolog\Formatter\JsonFormatter` for structured JSON logs (CloudWatch, Datadog, Papertrail)
- **JSON health route** — `/up` health endpoint gains JSON response support for richer status data
- **Cloudflare Email Service** — new mail driver for Cloudflare email routing

### Laravel 13 (March 2026) — Major Release

- **PHP 8.3 minimum** required, supports 8.4 and 8.5
- **Laravel AI / Boost MCP** — first-party MCP server for AI assistants
- **New PHP Attributes:** `#[Middleware]`, `#[Authorize]` (controllers), `#[Group]`, `#[TestProperty]`, `#[UnitTest]` (testing), `#[Job]` family (queues)
- **Queue Routing** — `Queue::route()` for centralized job routing
- **assertStatus() fully removed** — use `assertSuccessful()` or `assertStatus(200)`

### Laravel 12 (February 2025)

- **Application Starter Kits** — React, Svelte, Vue, and Livewire with Shadcn components
- **PHP 8.2 minimum** required (8.3 recommended, 8.4, 8.5 supported)
- **Bootstrap 5 by default** — Bootstrap 4 no longer default
- **Vite as default bundler** — Laravel Mix officially deprecated
- **Health endpoint at `/up`** — built into framework
- **Per-second rate limiting** — `RateLimiter::for("api", ...)` supports `->perSecond()`
- **`php artisan serve` accepts `--host` and `--port`** natively
- **Breaking:** `App\Http\Kernel` and `App\Console\Kernel` fully removed
- **Breaking:** `config/app.php` no longer has `aliases` array

Sources: [Laravel 13 Release Notes](https://laravel.com/docs/13.x/releases) | [GitHub Releases](https://github.com/laravel/framework/releases) | [Packagist](https://packagist.org/packages/laravel/framework)
---
name: Laravel
slug: laravel-developer
version: 1.0.0
description: Production-grade Laravel development — ship robust apps without common pitfalls.
metadata: {"emoji":"🟠","requires":{"bins":["php","composer"]},"os":["linux","darwin","win32"]}
---

## Navigation

| Topic | File | When to Use |
|---|---|---|
| Models, relationships, query builder, migrations | `eloquent.md` + `migrations.md` | Any DB work |
| Routing, validation, request lifecycle | `controllers.md` | Building endpoints |
| Sanctum, policies, gates, CSRF | `auth.md` | User auth & permissions |
| Jobs, workers, retries, failed jobs | `queues.md` | Background processing |
| Templates, components, XSS, slots | `blade.md` | Blade/frontend work |
| CLI commands, scheduling, tinker | `artisan.md` | DevOps & maintenance |
| REST APIs, JSON, rate limiting | `api.md` | API development |
| XSS, SQL injection, hardening | `security.md` | Security-critical code |
| Caching, indexing, eager loading | `performance.md` | Optimization |
| PHPUnit, factories, feature tests | `testing.md` | Test-driven dev |
| Production deployment, env, configs | `deployment.md` | Going live |
| File uploads, S3, presigned URLs | `file-uploads.md` | Media handling |
| Model lifecycle, observers, events | `observers.md` | Decoupled logic |
| DB schema, seeding, index management | `migrations.md` | Schema management |
| Log channels, structured logging, monitoring | `logging.md` | Debugging & monitoring |
| Translations, locales, pluralization | `localization.md` | i18n/l10n |
| Rules, Form Requests, custom validators | `validation.md` | Input validation |

## Critical Rules (Never Forget)

- **`env()` only in config files** — returns null after `config:cache`
- **Eager load relationships** — `with('posts')` or N+1 queries hit on every loop
- **`$fillable` or `$guarded`** — unprotected mass assignment = security hole
- **`findOrFail()` over `find()`** — null returns silently, crashes are better than silent bugs
- **Queue jobs serialize models as IDs** — re-fetch on process, model may be stale/deleted
- **`DB::transaction()` rolls back on exceptions only** — `exit`/timeout bypasses it
- **`{!! !!}` skips escaping** — XSS vector, use `{{ }}` by default
- **Middleware order matters** — earlier wraps later execution
- **`required` passes empty string** — use `required|filled` for actual content
- **`firstOrCreate` persists immediately** — `firstOrNew` lets you validate before saving
- **Route model binding defaults to `id`** — override `getRouteKeyName()` for slugs

## Version Defaults

- **Laravel 13** (latest — requires PHP 8.3+)
- PHP 8.3+ (8.2 minimum, optimized for 8.4)
- Composer 2.x
- SQLite for local dev (zero config)
- Vite for asset bundling (not Laravel Mix)
- Laravel Reverb for WebSocket/real-time

## Pro Tips

- Run `php artisan route:list --json` to get all routes as JSON for programmatic use
- Use `php artisan tinker` to test code in real Laravel context
- Use `php artisan about` to see full environment summary
- Use `php artisan optimize` after major changes in production
- Use `php artisan make:model Post -mcr` to generate Model + Migration + Controller in one shot

Start with `eloquent.md` for most common Laravel work.
