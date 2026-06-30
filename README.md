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
| **Laravel 13** | ✅ Latest (v13.17.0, June 23 2026 — still current) | 8.3+ | AI/Boost MCP, Reverb, new attributes, PHP 8.4/8.5 |
| Laravel 12 | Stable | 8.2+ | Bootstrap 5, Vite, per-second rate limiting |
| Laravel 11 | Security fixes | 8.1+ | Cleaner bootstrap, health endpoint |

---

## 🔄 Auto-Updater

The skill is **auto-updated every 6 hours** via a cron job. The agent decides whether to update based on whether it finds meaningful gaps — no forced updates.

**How it works:**
1. Checks current state of all 18 topic files
2. Identifies weak or outdated areas via web research
3. Checks for new Laravel/dependency releases
4. Updates files with new patterns + source URLs
5. Commits and pushes to `main` branch automatically

**Last research cycle:** 2026-06-30 06:20 UTC (cycle 13) — Laravel 13.17.0 still latest, no new framework release, no new framework CVEs (CVE-2026-39976 Passport auth bypass, CVE-2026-48019 CRLF email injection, CVE-2026-48555/48557 Spatie SSRF, CVE-2026-49287 Statamic, and the Laravel-Lang supply chain compromise from May 22 are all already covered). Filled the **Eloquent Strictness Hardening** gap in `eloquent.md`: previously only `Model::preventLazyLoading()` was mentioned (line 26) — added `preventSilentlyDiscardingAttributes()` (9.28+), `preventAccessingMissingAttributes()` (9.32+), and `prohibitDestructiveCommands()` (11+) as a consolidated section, all four promoted heavily at Laravel Live UK 2026 on June 29. These convert silent Eloquent bugs (typo'd attribute, missing `$fillable`, accidental mass-delete) into loud exceptions in dev. Recommended `AppServiceProvider::boot()` block + production-mode tuning (lazy-load log-to-Sentry, but throw on silent-discard/missing-attribute) included. SKILL.md bumped v1.22.0 → v1.22.1 (minor doc-add).

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
- **~9,600 lines** of production-ready content
- **Update cadence:** Every 6 hours via OpenClaw cron — 10 cycles in last 2 days (each targeting the oldest untouched files or new CVEs)
- **Auto-updated** via OpenClaw cron — never stale
- **MIT licensed** — free for everyone

---

<p align="center">
  <a href="https://github.com/clawvpsai/laravel-skill">📂 GitHub Repo</a>
  · <a href="https://laravel.com/docs">📖 Laravel Docs</a>
  · <a href="https://laravelversions.com">📋 Laravel Versions</a>
</p>