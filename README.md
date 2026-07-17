# 🟠 Laravel Developer Skill

> Production-grade Laravel development skill for OpenClaw agents — ship robust apps without common pitfalls.

[![Laravel 13](https://img.shields.io/badge/Laravel-13-blue?style=flat-square)](https://laravel.com)
[![PHP 8.3+](https://img.shields.io/badge/PHP-8.3+-purple?style=flat-square)](https://php.net)
[![MIT](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)
[![Auto-updated](https://img.shields.io/badge/Auto--updated-6h-blue?style=flat-square)](#auto-updater)

> **Latest tracked version:** Laravel **13.20.0** + **12.64.0** (July 14, 2026) — both tagged same day. Cycle 36: localization.md gap-fill — Lang::handleMissingKeysUsing (Laravel 10.33+) + Lang::hasForLocale vs Lang::has (10-yr footgun) + Carbon isoFormat vs format vs translatedFormat + word-order placeholder pattern (DE/AR/HI/JP) + spatie/laravel-translatable Eloquent attribute translation (v6.x) + Translator::addPath/addJsonPath/addNamespace for multi-tenant translation paths (cycle 35: ai.md JsonSchema anyOf + union types) (cycle 34: file-uploads.md gap-fill — Storage::fake vs persistentFake decision matrix + putFile/putFileAs/writeStream vs storeAs decision matrix + S3 multipart >5 GB pattern + temporaryUploadUrl driver restrictions (s3 + local only) + finfo Octane hygiene) (cycle 33: observers.md gap-fill — ShouldQueue on observers + ShouldHandleEventsAfterCommit + Octane state leak + withoutEvents/updateQuietly/saveQuietly decision matrix + Event::fake patterns + setObservableEvents) (Cycle 38: security.md CVE update — plank/laravel-mediable 7.0.0 shipped (Jul 12-13) closing CVE-2026-49969 SSRF via RemoteUrlAdapter + CVE-2026-49970 path traversal via File::sanitizePath(); previous CVE-2026-4809 entry marked 'No Patch' was stale — section now reflects patched status with SSRF defense-in-depth PHP pattern) (Cycle 37: artisan.md gap-fill — $this->components->bulletList() + the components output factory + schedule:clear-cache (NOT schedule:clear) for stuck withoutOverlapping() mutexes + Artisan::call() exit-code propagation under Octane (state leak + Kernel reuse + BufferedOutput capture) + php artisan down flag reference (--retry/--refresh/--render/--redirect/--secret cross-link to deployment.md)) (Cycle 39: blade.md gap-fill — @class and @style directives (Laravel 9.18+, the most-missed Blade directive) + <x-slot:foo> short-form slot syntax (Laravel 9+) + @aware directive for parent-to-child data flow without explicit prop threading + <x-dynamic-component> for runtime-resolved component names (CMS widgets, A/B tests, plugin systems) + view:cache/view:clear + Octane stale view cache — view:cache is NOT part of `php artisan optimize`, must be added to deploy scripts explicitly)
> **PHP baseline:** 8.3.32 / 8.4.23 / 8.5.8 (all security fixes as of 2026-07-01 batch)
> **Skill version:** v1.22.27 (41 auto-update cycles since 2026-06-28)

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
| **Laravel 13** | ✅ Latest (v13.20.0, July 14 2026) | 8.3+ | AI/Boost MCP, Reverb, new attributes, PHP 8.4/8.5; **13.20.0** adds `Illuminate\Image` first-party image processing (Intervention Image v4), `#[WithoutMiddleware]` attribute, `incrementEachQuietly`/`decrementEachQuietly` Eloquent bulk helpers, `QueueFake::beforePushing`/`afterPushing`, `WorkerStopping::$memory`, Redis session prefix, `Storage::assertEmpty()`; 13.19.0 shipped Collection::reduceInto, Str::counted, Http::query(), bulk SQS via SendMessageBatch |
| Laravel 12 | Stable (v12.64.0, July 14 2026) | 8.2+ | Bootstrap 5, Vite, per-second rate limiting; 12.63.0 backports PG whereDate/whereTime Expression fix (PR #60540), cache lock refresh (PR #58349), and lost-connection error messages (PR #60472). EOL bug fixes **Aug 13, 2026** |
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

**Last research cycle:** 2026-07-17 18:00 UTC (cycle 41)

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
- **~11,140 lines** of production-ready content
- **Update cadence:** Every 6 hours via OpenClaw cron — 41 cycles in last 17 days (each targeting the oldest untouched files or new CVEs)
- **Auto-updated** via OpenClaw cron — never stale
- **MIT licensed** — free for everyone

---

<p align="center">
  <a href="https://github.com/clawvpsai/laravel-skill">📂 GitHub Repo</a>
  · <a href="https://laravel.com/docs">📖 Laravel Docs</a>
  · <a href="https://laravelversions.com">📋 Laravel Versions</a>
</p>