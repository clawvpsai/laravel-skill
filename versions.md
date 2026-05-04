# Laravel Version-Specific Requirements

## Active LTS Versions

- **Laravel 10** — Security fixes only, no new features
- **Laravel 11** — Previous stable
- **Laravel 12** — Current latest (released February 2025)

## Version Selector Prompt

When working with Laravel, start by determining the version. Ask the user or check `composer.json`:
```bash
grep '"laravel/framework"' composer.json | grep -oP '\d+\.\d+'
```

Then load the relevant version sections below.

---

## Laravel 12 (Latest — February 2025)

### New in Laravel 12

- **Application Starter Kits** — React, Svelte, Vue, and Livewire with Shadcn components
- **PHP 8.4 support** — optimized for PHP 8.4 features (property hooks, asymmetric visibility)
- **PHP 8.2 minimum required** — officially supports 8.2, 8.3, 8.4
- **Bootstrap 5 by default** — no more Bootstrap 4
- **Vite as default bundler** — Laravel Mix officially deprecated
- **Health endpoint at `/up`** — built into framework, no route needed
- **`php artisan serve` accepts `--host` and `--port`** natively
- **`Queue::after()` and `Queue::failure()` methods** — cleaner queue events
- **Per-second rate limiting** — `RateLimiter::for('api', ...)` supports `perSecond()`
- **`Str::stripTags()`** — better than `strip_tags()` PHP原生
- **Carbon 3** — if upgrading, watch for `startOfDay()` / `endOfDay()` behavior changes
- **Laravel Reverb** — first-party WebSocket server for real-time apps
- **"Relatively minor maintenance release"** — most apps upgrade without code changes

### Breaking Changes from 11

- `bootstrap/app.php` no longer registers all service providers by default
- `config/app.php` no longer has `aliases` array — uses individual facades natively
- Route middleware registered in `bootstrap/app.php` via `$middleware->` not Kernel.php
- `App\Http\Kernel` and `App\Console\Kernel` fully removed

### Migration to Laravel 12

```bash
composer require laravel/framework:^12.0 --with-all-dependencies
php artisan --version  # verify
```

---

## Laravel 11 (2024 — Previous Stable)

### New in Laravel 11

- **Application skeleton cleaned** — far less boilerplate
- **`bootstrap/app.php`** replaces `Kernel.php` for middleware registration
- **`config/app.php`** much shorter — providers and aliases removed by default
- **Per-second rate limiting** via `RateLimiter::for()` with `->perSecond()`
- **Health check endpoint** — `Route::get('/up', fn() => 'ok')`
- **`php artisan about`** shows comprehensive environment summary
- **`DB::table()` query logging** improved
- **Password rehashing** — happens automatically on login
- **`assertSuccessful()`** on responses (was `assertStatus(200)`)

### Breaking Changes from 10

- `App\Http\Kernel` removed — use `bootstrap/app.php`
- `App\Console\Kernel` removed — use `routes/console.php`
- Route middleware registers in `bootstrap/app.php`
- `config/cors.php` — now via `withMiddleware` CORS config

### Migration to Laravel 11

```bash
composer require laravel/framework:^11.0 --with-all-dependencies
```

**Key file changes:**
- `bootstrap/app.php` — now registers routing + middleware chains
- `Kernel.php` files (Http/Console) — no longer exist, absorbed into bootstrap
- `routes/web.php` — now only web routes, API routes in `routes/api.php`
- `config/` — many config files slimmed down

---

## Laravel 10 (2023 — LTS, Security until late 2026)

### New in Laravel 10

- **PHP 8.1 minimum required**
- **`Illuminate/Validation/Rules/ProhibitedBy`** — opposite of `Rules/Unique`
- **Native rate limiting** — `RateLimiter` built into `RouteServiceProvider`
- **`RateLimiter::for()`** accepts callable returning `Limit` objects
- **Scheduled tasks can run on one server** — `onOneServer()` method
- **`assertSessionHas()`** no longer crashes on missing key — returns self for chaining
- **`Model::preventsAccessingMissingAttributes()`** — new lazy-loading guard
- **`Queue::except()`** — cancel queue processing for specific jobs
- **Tests use PHPUnit 10** — some method signatures changed

### Breaking Changes from 9

- PHP 8.0 no longer supported (8.1+ required)
- `assertStatus()` no longer accepts non-integer
- `Route::middleware()` — group-level middleware syntax simplified

### Migration to Laravel 10

```bash
composer require laravel/framework:^10.0 --with-all-dependencies
php artisan --version
```

---

## Version-Agnostic Best Practices (Always Apply)

### Always Use

- PHP 8.2+ (Laravel 12 requires 8.2+, optimized for 8.4)
- Composer 2.x
- Vite (not Mix — Mix deprecated in 11+)
- SQLite for local dev (zero config, no Docker needed)
- `php artisan key:generate` on fresh install

### Deprecation Timeline

| Feature | Deprecated In | Removed In |
|---|---|---|
| Laravel Mix | 11 (use Vite) | 12 |
| `assertStatus()` | 10 | 11 |
| `php artisan make:model -mcr` shorthand | Still works | — |
| `bootstrap/app.php` Kernel pattern | 11 (old style) | 12 |
| `config/app.php` aliases array | 11 | 12 |
| PHP 8.0/8.1 support | 12 | — |
| Bootstrap 4 default | 12 | — |

### Version Detection Quick Reference

```bash
# Check Laravel version
php artisan --version
# Laravel 12.x.x

# Check PHP version
php -v
# PHP 8.4.x

# Check composer
composer --version
# Composer 2.8.x
```

### Always Ask When Unknown

If working on a Laravel project and version is unknown, check:
1. `composer.json` — `"laravel/framework": "^12.0"` or `^11.0` etc.
2. `bootstrap/app.php` — if it has `Application::configure()` = Laravel 11+
3. `app/Http/Kernel.php` — if this file exists = Laravel 10 or older

---

## Agent Checklist Per Task

Before working on any Laravel task:
- [ ] Identify Laravel version
- [ ] Check PHP version compatibility (8.2+ for 12)
- [ ] Load version-specific rules above
- [ ] Check if feature is version-specific (API, queue syntax, etc.)
- [ ] Apply version-specific patterns, not generic ones