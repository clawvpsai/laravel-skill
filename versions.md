# Laravel Version-Specific Requirements

## Active Versions

- **Laravel 11** — Still receives security fixes
- **Laravel 12** — Active development
- **Laravel 13** — Current latest (v13.9.0 as of May 2026)

## Version Selector Prompt

When working with Laravel, start by determining the version. Ask the user or check `composer.json`:
```bash
grep '"laravel/framework"' composer.json | grep -oP '\d+\.\d+'
```

Then load the relevant sections below.

---

## Laravel 13 (Latest — May 2026, v13.9.0)

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

### New in Laravel 13.9 (May 2026)

- **PreparesForDispatch interface for Jobs** — new `PreparesForDispatch` interface lets jobs execute preparation logic before being dispatched to the queue. Useful for validation, enrichment, or state checks that run synchronously at dispatch time rather than async in handle().
- **PendingDispatch conditionable** — `PendingDispatch` now supports `->when()` and `->unless()` conditionals for cleaner conditional dispatch logic.
- **Migration events with name** — `MigrationStarted` and `MigrationEnded` events now carry the migration name for more granular event handling.
- **Generic return types on Builder paginate** — `paginate()`, `simplePaginate()`, `cursorPaginate()` now return properly typed `LengthAwarePaginator`/`Paginator` instances.
- **Queue:pause error display** — `php artisan queue:pause` now shows a clear error when the Worker is not pausable.
- Internal: `mt_rand()` replaces `rand()`, `preg_split` replaces `mb_split`, PSR-7 multipart array handling fixed.

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

---

Sources: [Laravel 13 Release Notes](https://laravel.com/docs/13.x/releases) | [GitHub Releases](https://github.com/laravel/framework/releases) | [Packagist](https://packagist.org/packages/laravel/framework)
