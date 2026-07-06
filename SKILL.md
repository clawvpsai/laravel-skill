---
name: Laravel
slug: laravel-developer
version: 1.22.14
description: Production-grade Laravel development — ship robust apps without common pitfalls.
metadata:
  {"emoji":"🟠","requires":{"bins":["php","composer"]},"os":["linux","darwin","win32"]}
---

## Navigation

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
| Production deployment, env, configs | `deployment.md` | Going live |
| File uploads, S3, presigned URLs | `file-uploads.md` | Media handling |
| Model lifecycle, observers, events | `observers.md` | Decoupled logic |
| DB schema, seeding, index management | `migrations.md` | Schema management |
| Log channels, structured logging, monitoring | `logging.md` | Debugging & monitoring |
| Composer self-update, CVE-2026-45793 / 40176 / 40261, supply chain | `security.md` (Composer section) + `deployment.md` step 0 | Updating Composer in deploy pipeline |
| Translations, locales, pluralization | `localization.md` | i18n/l10n |
| Rules, Form Requests, custom validators | `validation.md` | Input validation |
| Prohibited rule family + `Rule::when()` / `Rule::unless()` | `validation.md` (Prohibited Family + Rule::when sections) | Either-or fields, conditional rule chains, "missing/empty" enforcement |
| `#[Boot]` / `#[Initialize]` model lifecycle attributes | `observers.md` (model attribute hooks section) | Per-instance defaults + class-level event listeners without `boot()` override |
| `#[FailOnUnknownFields]` Form Request attribute | `validation.md` (Failing on Unknown Fields section) | Reject unknown fields at validation time (defense in depth) |
| `Context` facade for log context | `logging.md` (Context facade section) | Request/job-scoped context that auto-attaches to logs and survives queue dispatch |
| `Password::toPasswordRulesString()` | `validation.md` (Password rules section) | Derive JS `passwordrules` HTML hint from server-side `Password::` rule |
| `Features::passkeyAuthentication()` / `passkeyRegistration()` | `auth.md` (Starter Kits section) | Laravel 13 starter-kit + Fortify passkey integration |
| `#[UsePolicy]` model attribute (Laravel 13) | `auth.md` (Policy Registration section) | Colocate policy with model — replace `Gate::policy()` boot code |
| `#[Authorize]` controller attribute (Laravel 13) | `auth.md` (Controller Authorization Attributes section) + `controllers.md` | Declarative policy check on controller methods, route-param resolution built-in |
| `assertInvalid()` / `assertValid()` / `assertOnlyInvalid()` | `testing.md` (Validation Assertions section) | Laravel 11+ generic validation assertions for JSON + session flows |
| `Exceptions` facade (`assertReported`, `assertNothingReported`, `throwFirstReported`) | `testing.md` (Exception Assertions section) | Laravel 11.5+ clean exception assertions without try/catch |
| Pest 3 architecture testing (`->arch()->preset()->laravel()`, `->toBeFinal()`, `->toHaveMethodsDocumented()`) | `testing.md` (Pest 3 section) | Enforce Laravel conventions + design rules in CI |
| Pest 3 mutation testing (`pest --mutate`) | `testing.md` (Pest 3 section) | Verify tests actually catch injected bugs |
| Laravel AI SDK vs Laravel MCP vs Laravel Boost | `ai.md` (matrix section) | Pick the right product: AI SDK = app calls AI; MCP = expose app as MCP server; Boost = dev-time AI assistant context |
| Single action controllers (`__invoke`) | `controllers.md` (Single Action Controllers section) | Webhook handlers, OIDC callbacks, billing redirects — actions that don't fit the 7-method resource model |
| `Route::pattern()` global constraints | `controllers.md` (Route Patterns section) | Force `{id}` to match `\d+` (or any regex) app-wide; watch 404 vs `ModelNotFoundException` in tests |
| HEAD request cache headers (PR #60589) | `controllers.md` (API Versioning section) | 13.18.0+ finally sets `Cache-Control` / `ETag` on HEAD; CDN cache-warming scripts broke before |
| `WorkerStopping` `jobsProcessed` + `lastJobProcessedAt` (PR #60592) | `queues.md` (WorkerStopping section) | Lifetime-job-count and graceful-shutdown-gap dashboards without dirty reflection hacks |
| `php artisan dev` priority-based registration (PR #60580) | `artisan.md` (Dev Orchestration section) | 13.18.0+ lets vendor packages register a deterministic order so critical dev processes (log tail, queue worker) come first |
| `php artisan dev --kill-others-on-fail` (PR #60606) | `artisan.md` (Dev Orchestration section) | 13.18.0+ tears down sibling dev processes on non-zero exit; use in CI / one-shot dev, leave off for normal local dev |
| TaggedCache `flexible()` lock/defer namespace fix (PR #60626) | `performance.md` (TaggedCache section) | 13.18.0+ namespaces `flex_lock:` and `flex_defer:` keys separately so a custom lockName can't collide with a defer label |
| Debounced-jobs cache-hit reduction (PR #60575) | `performance.md` (Cache debounce subsection) | 13.18.0+ skips lock acquisition when already inside the debounce window — pairs with the 13.17.0 `maxWait` fix |
| `Release` queue middleware (`Illuminate\Queue\Middleware\Release`) (PR #60630) | `queues.md` (Job Middleware section) | 13.18.1+ declarative `->middleware(new Release($delay))` so jobs release themselves without `$this->release()` calls scattered through `handle()` |
| `$this->input()` on console commands (PR #60607) | `artisan.md` (Defining Commands section) | 13.18.1+ typed `$this->input('email')` / `$this->input(['email', 'name'])` — cleaner than `$this->argument()` + `$this->option()` + manual casting |
| `assertDatabaseEmpty()` accepts iterables (PR #60621) | `testing.md` (Database Assertions section) | 13.18.1+ lets you write `$this->assertDatabaseEmpty(User::all())` or `[$tableA, $tableB]` — was string-only and threw on collections |
| `api` / `json` routes respect `php artisan down` (PR #60595) | `controllers.md` (Maintenance section) + `deployment.md` (Maintenance Mode section) | 13.18.1+ returns JSON `503` for API/JSON callers under maintenance with `secret` bypass — drop hand-rolled maintenance middleware for API routes |
| `on-demand` log stacks respect configured channel name (PR #60635) | `logging.md` (Stacked / On-Demand Channels section) | 13.18.1+ `Log::build(['driver' => 'errorlog', 'channel' => 'audit'])` writes to the `audit` channel — was silently falling through to parent in some callsites |
| Inspect delayed jobs on `Queue::fake()` (PR #60636) | `testing.md` (Queue Assertions section) | 13.18.1+ queue fake tracks `availableAt` so `assertPushed(Job::class)` followed by `->delay(60)` inspections work — was lost on dispatch |
| `Str::mask()` encoding-aware tail (PR #60646) | `validation.md` (Sanitization Helpers section) + `security.md` (Output Encoding section) | 13.18.1+ doesn't chop a multi-byte character at the mask boundary — was emitting partial UTF-8 characters on long-tail strings |
| `foreignUuid` / `foreignUlid` Blueprint return types (PR #60643) | `migrations.md` (Foreign Key Constraints section) | 13.18.1+ returns `ColumnType::Uuid` / `ColumnType::Ulid` matching `foreignId()` — schema-dumpers and migration generators handle them uniformly |
| Predis retry config accepts scalar values (PR #60642) | `deployment.md` (Redis Configuration section) | 13.18.1+ `redis.options.read_write_timeout` etc. can be scalars in `config/database.php` without breaking `config:cache` (Predis-only; PhpRedis unaffected) |
| `RateLimited` middleware `releaseAfter` `__sleep()` fix (PR #60609) | `queues.md` (RateLimited middleware section) | 13.18.0+ serializes throttle timestamps so `dispatch($job->afterCommit())` doesn't lose the throttle window |
| `schedule:work` graceful signal handling (PR #60616) | `artisan.md` (schedule:work section) | Long-lived scheduler in Docker/K8s/supervisord exits cleanly on SIGTERM, lets in-flight runs finish |
| Soft-delete `restore()` event gating (PR #60605) | `observers.md` (restored callback note) | Laravel 13.17+ only fires `restored` when the underlying save() actually succeeded — check the boolean return |
| Embedding caching (`Embeddings::for(...)->cache(seconds: ...)`) | `ai.md` (Embedding Caching section) | Skip duplicate API calls for repeated inputs (knowledge base, FAQs) |
| `->queue()` for long-running AI calls | `ai.md` (Queue Long-Running AI Calls section) | Audio transcription, image gen, large-doc summarization — don't block HTTP |
| Prompt caching (`providerOptions(['cache_control' => [...]])`) | `ai.md` (Prompt Caching section) | Anthropic / OpenAI 90% discount on cached input tokens for agents with fixed system prompts |
| `RemembersConversations` trait + `Conversational` interface | `ai.md` (Conversation Memory section) | Multi-turn chat that persists history automatically — required for any real chatbot / support agent / in-product assistant |
| Per-class `Agent::fake()` + `preventStrayPrompts()` + `assertPrompted()` | `ai.md` (Testing AI Agents section) | Zero-API-cost AI tests in CI; per-agent scoping is cleaner than facade-level `AI::fake()` |
| Per-resource fakes (`Image::fake`, `Transcription::fake`, `Embeddings::fake`, `Reranking::fake`, `Files::fake`) | `ai.md` (Testing AI Agents section) | Each AI resource has its own fake + assertions — `AI::fake()` only covers text completions |
| `HasStructuredOutput` interface + `schema()` auto-fake | `ai.md` (Testing AI Agents section) | Structured-output fakes auto-generate schema-matching fake data — no hand-written JSON |
| `php artisan i18n:check` (missing translation detector) | `localization.md` (Detecting Missing Translations section) | CI integration to fail builds on incomplete translations |
| Lazy translation loading (JSON-only, lazy namespaces, DB+cache) | `localization.md` (Lazy Translation Loading section) | Avoid loading thousands of keys on every request for large apps |
| `Model::preventLazyLoading()` / `preventSilentlyDiscardingAttributes()` / `preventAccessingMissingAttributes()` / `prohibitDestructiveCommands()` | `eloquent.md` (Eloquent Strictness Hardening section) | Turn silent Eloquent bugs into loud exceptions in dev — N+1, missing `$fillable`, typos, accidental mass-delete |
| `Number::forHumans` / `Number::abbreviate` / `Number::fileSize` INF/NaN guard (PR #60617 + #60625) | `performance.md` (Number OOM section) + `api.md` (Number DoS section) | 13.18.0+ stops these from recursing on `INF`/`NaN` and OOM-crashing the worker — treat as a remote-DoS vector on API responses that render user input |
| `Request::json()` top-level zero body fix (PR #60614) | `api.md` (Error Responses section) | 13.18.0+ preserves a literal `0` request body (was coerced to `[]`); `PUT /counters/abc/reset` endpoints with body `0` now work without a `?? 0` workaround |
| `@checked` / `@selected` / `@disabled` / `@readonly` / `@required` form-helper directives | `blade.md` (Form-Helper Directives section) | Collapse the verbose `{{ old(...) == $x ? 'checked' : '' }}` pattern into single conditional attributes; emits static attribute when truthy, nothing when falsy (also XSS-safer than hand-rolled concatenations) |
| `@verbatim` directive (Vue/Alpine/JS) | `blade.md` (`@verbatim` section) | Wrap a chunk of frontend-framework template syntax so Blade doesn't try to parse `{{ }}` and `@click`; can't be nested |
| `Blade::render()` inline string rendering | `blade.md` (Rendering Inline Blade Strings section) | AI-generated output, CMS user templates, DB-stored email templates; **NOT sandboxed** — user-supplied templates can RCE via `@php` or arbitrary method calls; pass `deleteCachedView: true` and ideally whitelist directives or use Twig/Mustache for user-facing templates |
| Inline component views via `<<<'blade'` heredoc | `blade.md` (Inline Component Views section) | Class-based components return the Blade template directly from `render()`; best for small components, >20 lines still want external view files for cache + IDE highlighting |
| FrankenPHP + Octane (Caddy single-binary runtime) | `deployment.md` (FrankenPHP section) | Laravel Cloud's underlying runtime; HTTP/3 + auto HTTPS + Mercure + Brotli out of the box; preferred over Swoole/RoadRunner for new Octane deploys on PHP 8.3+ |
| OPcache preload + tracing JIT (PHP 8.3+) | `deployment.md` (OPcache + JIT section) | Single biggest free performance win — preload the framework autoloader into shared memory at worker boot; pair with `validate_timestamps=0` + SAPI reload on deploy |
| FrankenPHP thread-pool split (slow-endpoint isolation) | `deployment.md` (FrankenPHP section) | Mirror of FPM `pm` pool split — keep `/api/reports/*` and `/api/exports/*` on a dedicated thread pool so they can't starve the main request pool |
| `LARAVEL_STORAGE_PATH` for embedded FrankenPHP binaries | `deployment.md` (FrankenPHP section) | Each new binary extraction lives in a different temp dir — set `LARAVEL_STORAGE_PATH=/var/lib/yourapp/storage` or call `Application::useStoragePath()` so uploads/logs/caches survive upgrades |


## Critical Rules (Never Forget)

- **`env()` only in config files** — returns null after `config:cache`
- **Eager load relationships** — `with('posts')` or N+1 queries hit on every loop
- **`Model::preventX()` strictness hooks** — every Laravel project should turn on `preventSilentlyDiscardingAttributes()`, `preventAccessingMissingAttributes()`, and `preventLazyLoading()` in `AppServiceProvider::boot()`. Default Laravel skeleton ships with none.
- **`$fillable` or `$guarded`** — unprotected mass assignment = security hole
- **`findOrFail()` over `find()`** — null returns silently, crashes are better than silent bugs
- **Queue jobs serialize models as IDs** — re-fetch on process, model may be stale/deleted
- **`DB::transaction()` rolls back on exceptions only** — `exit`/timeout bypasses it
- **`{!! !!}` skips escaping** — XSS vector, use `{{ }}` by default
- **`new HtmlString($userInput)` also bypasses escaping** — class-based `{!! !!}`. Always `e($value)` before wrapping (CVE-2026-33080 in Filament was a real-world instance of this pattern)
- **Middleware order matters** — earlier wraps later execution
- **`required` passes empty string** — use `required|filled` for actual content
- **`firstOrCreate` persists immediately** — `firstOrNew` lets you validate before saving
- **Route model binding defaults to `id`** — override `getRouteKeyName()` for slugs
- **`Route::metadata()` survives `route:cache`** (Laravel 13.17+) — first-class route metadata via dot notation; use for audit, feature flags, OpenAPI emitters
- **`Agent::fake()` belongs in every AI test** — never hit a real LLM in CI. Add `->preventStrayPrompts()` so any unmocked agent call throws (catches newly-added agent invocations before they leak API costs into CI).
- **Don't pair `messages()` with `RemembersConversations`** — defining `messages()` manually makes the trait silently no-op. Pick ONE: trait = automatic DB-backed history, `messages()` = you own storage (Redis, custom scoped table, etc.).
- **`schedule:work` now catches signals** (Laravel 13.17+, PR #60616) — `SIGINT`/`SIGTERM`/`SIGQUIT` stop new runs cleanly and let in-flight `schedule:run` finish; long-lived scheduler processes no longer need wrapper scripts
- **HEAD request cache headers** (Laravel 13.18.0+, PR #60589) — `SetCacheHeaders` middleware now applies to `HEAD`. Before: CDN cache-warm scripts got no `Cache-Control`/`ETag`. Force cache headers explicitly if stuck on 13.16

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