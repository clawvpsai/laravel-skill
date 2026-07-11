# 🟠 Laravel Developer Skill

> Production-grade Laravel development skill for OpenClaw agents — ship robust apps without common pitfalls.

[![Laravel 13](https://img.shields.io/badge/Laravel-13-blue?style=flat-square)](https://laravel.com)
[![PHP 8.3+](https://img.shields.io/badge/PHP-8.3+-purple?style=flat-square)](https://php.net)
[![MIT](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)
[![Auto-updated](https://img.shields.io/badge/Auto--updated-6h-blue?style=flat-square)](#auto-updater)

> **Latest tracked version:** Laravel **13.19.0** + **12.63.0** (July 7, 2026) — both tagged same day. Cycle 33: observers.md gap-fill — ShouldQueue on observers (ModelNotFoundException gotcha) + ShouldHandleEventsAfterCommit (GitHub #52440) + Octane state leak + withoutEvents/updateQuietly/saveQuietly decision matrix + Event::fake patterns + setObservableEvents (cycle 32: performance.md gap-fill — EXPLAIN/index composite order rules + cache stampede SWaR + lock patterns; cycle 31: Http::query, Collection::reduceInto, Str::counted, query/queryJson helpers, bulk SQS, assertSoftDeleted deletedAtColumn, DateRule past/future helpers, PG whereDate/whereTime fix)
> **PHP baseline:** 8.3.32 / 8.4.23 / 8.5.8 (all security fixes as of 2026-07-01 batch)
> **Skill version:** v1.22.19 (33 auto-update cycles since 2026-06-28)

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
| **Laravel 13** | ✅ Latest (v13.19.0, July 7 2026) | 8.3+ | AI/Boost MCP, Reverb, new attributes, PHP 8.4/8.5; 13.19.0 ships Collection::reduceInto (PR #60651), Str::counted (PR #60649), Http::query() (PR #60663), query/queryJson HTTP test helpers (PR #60662), bulk SQS via SendMessageBatch (PR #60645), assertSoftDeleted deletedAtColumn (PR #60657), DateRule past/future/nowOrPast/nowOrFuture (PR #60687), and PG whereDate/whereTime Expression fix (PR #60540) |
| Laravel 12 | Stable (v12.63.0, July 7 2026) | 8.2+ | Bootstrap 5, Vite, per-second rate limiting; 12.63.0 backports PG whereDate/whereTime Expression fix (PR #60540), cache lock refresh (PR #58349), and lost-connection error messages (PR #60472). EOL bug fixes **Aug 13, 2026** |
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

**Last research cycle:** 2026-07-05 18:11 UTC (cycle 26) — No new Laravel framework release (13.18.1 still head of 13.x, 3 days old). `ai.md` was the oldest untouched file (6 days stale) and was gap-filled with the two most important missing topics for production AI work: **(1) Conversation Memory** — `Conversational` attribute + `RemembersConversations` trait for zero-config auto-persisted multi-turn chat (backed by `agent_conversations` + `agent_conversation_messages` tables from the AI SDK migration), plus the manual `messages()` override path for custom storage (Redis, tenant-scoped tables) — including the critical docs warning that defining `messages()` AND using the trait makes the trait silently no-op. **(2) Per-class testing fakes** — `Agent::fake([...])` with FIFO canned responses, `Agent::fake(closure)` for dynamic responses, `->preventStrayPrompts()` to catch unmocked calls in CI, and `assertPrompted()` / `assertNeverPrompted()` for assertions. Plus per-resource fakes the skill was missing: `Image::fake(closure)`, `Transcription::fake([...])->preventStrayTranscriptions()`, `Embeddings::fake()` (auto-dim or explicit), `Reranking::fake([RankedDocument, ...])`, `Files::fake()`. Structured-output fakes now auto-generate schema-matching data when the agent implements `HasStructuredOutput`. Facade-level `AI::fake()` is now documented as the legacy path (still works, but per-class is preferred). SKILL.md bumped v1.22.12 → v1.22.13 — 4 new cross-reference entries for the new ai.md sections, 2 new Critical Rules (always-fake + messages() vs RemembersConversations conflict). 26 cycles in last 7 days.

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
- **Update cadence:** Every 6 hours via OpenClaw cron — 26 cycles in last 7 days (each targeting the oldest untouched files or new CVEs)
- **Auto-updated** via OpenClaw cron — never stale
- **MIT licensed** — free for everyone

---

<p align="center">
  <a href="https://github.com/clawvpsai/laravel-skill">📂 GitHub Repo</a>
  · <a href="https://laravel.com/docs">📖 Laravel Docs</a>
  · <a href="https://laravelversions.com">📋 Laravel Versions</a>
</p>