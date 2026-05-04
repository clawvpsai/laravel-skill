# Laravel Version-Specific Requirements

## Active LTS Versions

- **Laravel 11** — Still receives security fixes
- **Laravel 12** — Previous stable
- **Laravel 13** — Current latest (released March 2026)

## Version Selector Prompt

When working with Laravel, start by determining the version. Ask the user or check `composer.json`:
```bash
grep '"laravel/framework"' composer.json | grep -oP '\d+\.\d+'
```

Then load the relevant version sections below.

---

## Laravel 13 (Latest — March 2026)

### New in Laravel 13

- **PHP 8.3 minimum required** (8.2, 8.3, 8.4 supported)
- **Laravel AI / Boost MCP** — first-party MCP server for AI assistants
  - `/upgrade-laravel-v13` slash command for automated upgrades
  - Interactive HTML responses in Claude/VS Code Copilot
- **Improved Eloquent** — better attribute casting, faster queries
- **Faster routing** — optimized route resolution
- **Native TypeScript support** — improved scaffolding
- **Laravel Reverb** — first-party WebSocket server (production-ready)
- **"Relatively minor upgrade"** — most apps upgrade without code changes

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
- **PHP 8.2 minimum required** — supports 8.2, 8.3, 8.4
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
| `config/app.php` aliases array | 11 | 12 |
| PHP 8.0/8.1/8.2 support | 13 | — |
| Bootstrap 4 default | 12 | — |

### Version Detection

```bash
# Check Laravel version
php artisan --version
# Laravel 13.x.x

# Check PHP version
php -v
# PHP 8.4.x

# Check composer
composer --version
# Composer 2.8.x
```

---

## Agent Checklist Per Task

Before working on any Laravel task:
- [ ] Identify Laravel version (`php artisan --version`)
- [ ] Check PHP version compatibility (8.3+ for Laravel 13)
- [ ] Load version-specific rules above
- [ ] Check if feature is version-specific
- [ ] Apply version-specific patterns, not generic ones