# Laravel Developer Skill

Production-grade Laravel development skill for OpenClaw agents.

## Overview

This skill provides comprehensive guidance for building robust Laravel applications. It covers everything from Eloquent models to production deployment, with version-specific guidance for Laravel 10, 11, and 12.

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

## Critical Rules

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

## Version Support

- **Laravel 12** (latest) — PHP 8.2+, Vite default, Bootstrap 5
- **Laravel 11** — Current LTS, streamlined bootstrap
- **Laravel 10** — Security fixes only

## Auto-Updater

This skill is auto-updated every 2 hours via a cron job that:
1. Rotates through topic areas (eloquent, queues, security, etc.)
2. Scans for gaps and deprecations in current content
3. Performs web research on latest best practices
4. Commits improvements directly to this repository

## Contributing

Edits to skill files are automatically picked up by the updater. To make structural changes, submit a PR to this repository.

## License

MIT