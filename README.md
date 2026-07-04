# 🟠 Laravel Developer Skill

> Production-grade Laravel development skill for OpenClaw agents — ship robust apps without common pitfalls.

[![Laravel 13](https://img.shields.io/badge/Laravel-13-blue?style=flat-square)](https://laravel.com)
[![PHP 8.3+](https://img.shields.io/badge/PHP-8.3+-purple?style=flat-square)](https://php.net)
[![MIT](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)
[![Auto-updated](https://img.shields.io/badge/Auto--updated-6h-blue?style=flat-square)](#auto-updater)

---

## Why This Skill Exists

Most AI coding assistants hallucinate Laravel patterns that are deprecated, version-mismatched, or just wrong in production. This skill is the ground truth — auto-updated, version-aware, and battle-tested from 7+ years of shipping Laravel apps.

---

## 🚀 Quick Start

```bash
# Check Laravel version
php artisan --version

# Load SKILL.md first for navigation
# Then dive into the topic you need
```

---

## 📚 Topics Covered (17 files)

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
| AI agents, tools, streaming, embeddings | `ai.md` | AI/LLM integration |
| PHPUnit, factories, feature tests | `testing.md` | Test-driven dev |
| Production deployment, Docker, Octane | `deployment.md` | Going live |
| File uploads, S3, presigned URLs | `file-uploads.md` | Media handling |
| Model lifecycle, observers, events | `observers.md` | Decoupled logic |
| Log channels, structured logging | `logging.md` | Debugging & monitoring |
| Translations, locales, pluralization | `localization.md` | i18n/l10n |
| Validation rules, Form Requests | `validation.md` | Input validation |

---

## ⚠️ Critical Rules (Never Forget)

| Rule | Why |
|---|---|
| `env()` only in config files | Returns null after `config:cache` |
| Always eager load with `with()` | N+1 queries kill performance |
| Use `$fillable` or `$guarded` | Unprotected mass assignment = security hole |
| Use `findOrFail()` over `find()` | Silent nulls are worse than crashes |
| Queue jobs serialize models as IDs | Re-fetch in handler — model may be stale |
| `DB::transaction()` rolls back on exceptions only | `exit`/timeout bypasses it |
| `{!! !!}` skips escaping | XSS vector, use `{{ }}` by default |
| `required` fails on empty string | Use `required|filled` for actual content |

---

## 📦 Version Support

| Version | Status | PHP | Key Features |
|---|---|---|---|
| **Laravel 13** | ✅ Latest (v13.18.0, June 30 2026) | 8.3+ | AI/Boost MCP, Reverb, new attributes, PHP 8.4/8.5; 13.18.0 ships the Number INF/NaN DoS guard (PR #60617 + #60625), HEAD cache-header fix (PR #60589), schedule:work signal handling (PR #60616), and soft-delete event gating (PR #60605) |
| Laravel 12 | Stable | 8.2+ | Bootstrap 5, Vite, per-second rate limiting |
| Laravel 11 | Security fixes | 8.1+ | Cleaner bootstrap, health endpoint |

---

## 🔄 Auto-Updater

The skill is **auto-updated every 6 hours** via a cron job. The agent decides whether to update based on whether it finds meaningful gaps — no forced updates.

**How it works:**
1. Checks current state of all 19 topic files
2. Identifies weak or outdated areas via web research
3. Checks for new Laravel/dependency releases
4. Updates files with new patterns + source URLs
5. Commits and pushes to `main` branch automatically

**Last research cycle:** 2026-07-04 00:00 UTC (cycle 22) — Laravel 13.18.1 patch release. **New head of 13.x is v13.18.1** (tagged 2026-07-02 18:36 UTC, missed by the 2026-07-03 12:05 UTC cycle 21 which only ran validation.md). Two genuinely useful new features: **(1) `Release` queue middleware** (`Illuminate\Queue\Middleware\Release`) — declarative `->middleware(new Release($delay))` replaces scattered `$this->release()` calls in `handle()` (PR #60630, queues.md); **(2) `$this->input()` on console commands** — typed CLI input reader parallel to `request()->input()` (PR #60607, artisan.md). Plus 6 bug fixes: `api`/`json` routes respect `php artisan down` (PR #60595 — drop hand-rolled maintenance middleware for API routes); `assertDatabaseEmpty()` accepts iterables (PR #60621); on-demand log stacks respect channel name (PR #60635); inspect delayed jobs on `Queue::fake()` (PR #60636); `Str::mask()` encoding-aware tail (PR #60646); `foreignUuid`/`foreignUlid` Blueprint return types (PR #60643); Predis retry scalar config (PR #60642, niche). 9 new cross-reference entries added to SKILL.md v1.22.9 → v1.22.10. `versions.md` active-versions line + heading updated; new "New in Laravel 13.18.1" section added before the 13.18.0 history note. 22 cycles in last 6 days. No new Laravel framework version (v13.18.0 still head of 13.x, ~3 days since v13.18.0). No new framework CVEs. **Two new sections added to `deployment.md` (oldest untouched file, 4 days stale):** **(1) OPcache + JIT Production Tuning (PHP 8.3+)** — full production-grade `opcache.ini` config with `validate_timestamps=0` + `opcache.preload` script that loads the entire Illuminate framework + compiled config into shared memory at worker boot, plus tracing JIT sizing guidance, deploy invalidation options (SAPI reload vs `opcache_reset()` vs `file_cache_fallback`), and a `/metrics/opcache` monitoring endpoint. **(2) FrankenPHP (Laravel Cloud's underlying runtime)** — first-class Octane driver built on Caddy with built-in HTTP/3 + auto HTTPS + Mercure + Brotli; full install + `octane:frankenphp` + production Caddyfile with thread-pool split for slow endpoints, comparison table against Swoole/RoadRunner, and the critical `LARAVEL_STORAGE_PATH` caveat for embedded-binary deploys. SKILL.md bumped v1.22.7 → v1.22.8. 20 cycles in last 3 days.

---

## 🛠️ Contributing

```bash
# Clone
git clone git@github.com:clawvpsai/laravel-skill.git
cd laravel-skill

# Edit any .md file — updater picks up changes
# For structural changes, submit a PR
```

All skill files are `.md` — no code generation needed. Just update patterns, add examples, or fix deprecations.

---

## 📊 Skill Stats

- **19 topic files** covering full Laravel development lifecycle
- **Version-aware** — Laravel 13, 12, 11 covered
- **~10,330 lines** of production-ready content
- **Update cadence:** Every 6 hours via OpenClaw cron — 19 cycles in last 3 days (each targeting the oldest untouched files or new CVEs)
- **Auto-updated** via OpenClaw cron — never stale
- **MIT licensed** — free for everyone

---

<p align="center">
  <a href="https://github.com/clawvpsai/laravel-skill">📂 GitHub Repo</a>
  · <a href="https://laravel.com/docs">📖 Laravel Docs</a>
  · <a href="https://laravelversions.com">📋 Laravel Versions</a>
</p>