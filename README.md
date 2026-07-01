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

**Last research cycle:** 2026-07-01 00:03 UTC (cycle 16) — **Laravel v13.18.0 released 2026-06-30 12:55 UTC** (16 PRs, all bug fixes — no breaking changes; missed by cycle 15 which ran 5 hours before the tag). `versions.md` got a full **"New in Laravel 13.18"** section covering every fix in the release: `schedule:work` graceful signal handling (PR #60616), `WorkerStopping` `jobsProcessed` + `lastJobProcessedAt` payload (PR #60592 + #60608), soft-delete `restore()`/`restoring` event gating (PR #60605), `HEAD` cache-header fix (PR #60589 — 13.16 regression), `RateLimited` middleware `__sleep()` fix (PR #60609), `Request::json()` top-level zero body fix (PR #60614), `Number::forHumans`/`abbreviate`/`fileSize` INF/NaN guard (PR #60617 + #60625 — treated as remote-DoS on API responses rendering user input), `php artisan dev` priority registration + `--kill-others-on-fail` (PR #60580 + #60606), TaggedCache `flexible()` lock/defer namespace fix (PR #60626), debounced-jobs cache-hit reduction (PR #60575), conditional return types (PR #60586 + #60591). All cross-referenced into `api.md`, `controllers.md`, `observers.md`, `performance.md`, `queues.md`, `artisan.md`. The pre-existing "Late-June 2026 Bug Fixes" section was reframed from "pending fixes" → "shipped-in-13.18 history trail". Corrected all `13.17.1+` → `13.18.0+` references in `api.md`, `controllers.md`, `performance.md`, `SKILL.md` (the fixes were predicted to ship in v13.17.1 but actually shipped in v13.18.0 — no v13.17.1 was tagged). SKILL.md bumped v1.22.3 → v1.22.4. 16 cycles in last 3 days.

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
- **~9,980 lines** of production-ready content
- **Update cadence:** Every 6 hours via OpenClaw cron — 16 cycles in last 3 days (each targeting the oldest untouched files or new CVEs)
- **Auto-updated** via OpenClaw cron — never stale
- **MIT licensed** — free for everyone

---

<p align="center">
  <a href="https://github.com/clawvpsai/laravel-skill">📂 GitHub Repo</a>
  · <a href="https://laravel.com/docs">📖 Laravel Docs</a>
  · <a href="https://laravelversions.com">📋 Laravel Versions</a>
</p>