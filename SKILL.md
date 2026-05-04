---
name: Laravel
slug: laravel-developer
version: 1.0.0
description: Production-grade Laravel development — ship robust apps without common pitfalls.
metadata:
  {"emoji":"🟠","requires":{"bins":["php","composer"]},"os":["linux","darwin","win32"]}
---

## Navigation

| Topic | File | When to Use |
|---|---|---|
| Models, relationships, query builder, migrations | `eloquent.md` | Any DB work |
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